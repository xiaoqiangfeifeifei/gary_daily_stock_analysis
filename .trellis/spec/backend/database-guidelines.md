# Database Guidelines

> Database patterns and conventions for this project.

---

## Overview

This project uses **SQLAlchemy 2.0+ ORM** with **SQLite** as the database engine. SQLite is chosen because it is built into Python's standard library, requiring no additional installation per `requirements.txt`.

- **ORM base**: `sqlalchemy.orm.declarative_base()`
- **Engine**: Created via `sqlalchemy.create_engine()`, stored as a singleton in `DatabaseManager`
- **Session factory**: `sessionmaker(bind=engine, autocommit=False, autoflush=False)`
- **No migration tool**: Tables are created via `Base.metadata.create_all(engine)`. Schema changes are applied manually or via the `create_all` call on startup.

All models are defined in `src/storage.py`. Database access is through the **Repository pattern** in `src/repositories/`.

---

## DatabaseManager Singleton

The `DatabaseManager` class in `src/storage.py` (line 774) is a thread-safe singleton that manages the database connection:

```python
# src/storage.py, line 774-793
class DatabaseManager(metaclass=_DatabaseManagerMeta):
    _instance: Optional['DatabaseManager'] = None
    _init_lock = threading.RLock()

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._initialized = False
        return cls._instance

    @classmethod
    def get_instance(cls) -> 'DatabaseManager':
        with cls._init_lock:
            if cls._instance is None:
                cls()
            return cls._instance

    @classmethod
    def reset_instance(cls) -> None:
        """Reset singleton (for testing)."""
        with cls._init_lock:
            if cls._instance is not None:
                if hasattr(cls._instance, '_engine') and cls._instance._engine is not None:
                    cls._instance._engine.dispose()
                cls._instance._initialized = False
                cls._instance = None
```

### Session acquisition

All code gets sessions via `DatabaseManager.get_session()`:

```python
# src/storage.py, line 989-1008
def get_session(self) -> Session:
    if not getattr(self, '_initialized', False) or not hasattr(self, '_SessionLocal'):
        raise RuntimeError(
            "DatabaseManager 未正确初始化。"
            "请确保通过 DatabaseManager.get_instance() 获取实例。"
        )
    session = self._SessionLocal()
    try:
        return session
    except Exception:
        session.close()
        raise
```

The `get_session()` method does NOT act as a context manager on its own. Callers must manually close or use the session as a context manager.

---

## Repository Pattern

Each domain has a Repository class in `src/repositories/<domain>_repo.py`. Repositories receive a `DatabaseManager` via constructor injection, defaulting to the singleton:

```python
# src/repositories/portfolio_repo.py, line 45-49
class PortfolioRepository:
    def __init__(self, db_manager: Optional[DatabaseManager] = None):
        self.db = db_manager or DatabaseManager.get_instance()
```

```python
# src/repositories/alert_repo.py, line 23-27
class AlertRepository:
    def __init__(self, db_manager: Optional[DatabaseManager] = None):
        self.db = db_manager or DatabaseManager.get_instance()
```

Services then compose repositories:

```python
# src/services/alert_service.py, line 97-102
class AlertService:
    def __init__(self, db_manager: Optional[DatabaseManager] = None):
        self.db = db_manager or DatabaseManager.get_instance()
        self.repo = AlertRepository(self.db)
```

---

## Model Definition Conventions

All models are defined in `src/storage.py` using `declarative_base()`:

```python
# src/storage.py, line 62
Base = declarative_base()

# src/storage.py, line 70-118
class StockDaily(Base):
    __tablename__ = 'stock_daily'

    id = Column(Integer, primary_key=True, autoincrement=True)
    code = Column(String(10), nullable=False, index=True)
    date = Column(Date, nullable=False, index=True)
    open = Column(Float)
    close = Column(Float)
    volume = Column(Float)
    data_source = Column(String(50))
    created_at = Column(DateTime, default=datetime.now)
    updated_at = Column(DateTime, default=datetime.now, onupdate=datetime.now)

    __table_args__ = (
        UniqueConstraint('code', 'date', name='uix_code_date'),
        Index('ix_code_date', 'code', 'date'),
    )
```

Conventions:
- **`__tablename__`** is always `snake_case`, descriptive (e.g., `stock_daily`, `portfolio_accounts`, `analysis_history`)
- **`id`** is always `Integer, primary_key=True, autoincrement=True`
- **`created_at`** always has `default=datetime.now`
- **`updated_at`** always has `default=datetime.now, onupdate=datetime.now`
- **`code`** field for stock codes is `String(10)`
- Foreign keys use `ForeignKey('parent_table.column')`
- Unique constraints and indexes are defined in `__table_args__`
- Every model has a `__repr__` method
- DTO-like models include a `to_dict()` method

---

## Query Patterns

All queries use SQLAlchemy 2.0 style with `select()`, not the legacy `session.query()`:

```python
# src/repositories/portfolio_repo.py, line 87-91
def list_accounts(self, include_inactive: bool = False) -> List[PortfolioAccount]:
    with self.db.get_session() as session:
        query = select(PortfolioAccount)
        if not include_inactive:
            query = query.where(PortfolioAccount.is_active.is_(True))
        rows = session.execute(query.order_by(PortfolioAccount.id.asc())).scalars().all()
        return list(rows)
```

```python
# src/repositories/alert_repo.py, line 39-41
def get_rule(self, rule_id: int) -> Optional[AlertRuleRecord]:
    with self.db.get_session() as session:
        return session.execute(
            select(AlertRuleRecord).where(AlertRuleRecord.id == rule_id).limit(1)
        ).scalar_one_or_none()
```

Key patterns:
- Always use `select(Model)` with `.where()`, `.order_by()`, `.limit()`
- Combine conditions with `and_()` and `or_()`
- Use `desc()` for descending order
- Use `func` for aggregate queries (`func.count()`, `func.max()`, etc.)
- Use `.scalars().all()` for multiple results, `.scalar_one_or_none()` for single optional result
- Always wrap in `with self.db.get_session() as session:`

### Deletes

```python
# src/repositories/alert_repo.py, line 57-61
def delete_rule(self, rule_id: int) -> bool:
    with self.db.get_session() as session:
        result = session.execute(delete(AlertRuleRecord).where(AlertRuleRecord.id == rule_id))
        session.commit()
        return bool(result.rowcount)
```

---

## Upsert Pattern (sqlite_insert)

For bulk upserts, use `sqlite_insert` with `on_conflict_do_update`:

```python
# src/storage.py, line 1752-1773
stmt = sqlite_insert(StockDaily).values(chunk)
excluded = stmt.excluded
session.execute(
    stmt.on_conflict_do_update(
        index_elements=['code', 'date'],
        set_={
            'open': excluded.open,
            'high': excluded.high,
            'low': excluded.low,
            'close': excluded.close,
            'volume': excluded.volume,
            'amount': excluded.amount,
            'pct_chg': excluded.pct_chg,
            'ma5': excluded.ma5,
            'ma10': excluded.ma10,
            'ma20': excluded.ma20,
            'volume_ratio': excluded.volume_ratio,
            'data_source': excluded.data_source,
            'updated_at': excluded.updated_at,
        },
    )
)
```

Important: SQLite has a per-statement bind-parameter limit. The codebase chunks upserts into batches of 50 records to stay within bounds:

```python
# src/storage.py, line 1727
_SQLITE_CHUNK = 50
for i in range(0, len(records), _SQLITE_CHUNK):
    chunk = records[i : i + _SQLITE_CHUNK]
```

---

## Transaction Handling

### Simple transactions

For most read/write operations, use `with self.db.get_session() as session:` followed by `session.commit()`:

```python
# src/repositories/alert_repo.py, line 29-35
def create_rule(self, fields: Dict[str, Any]) -> AlertRuleRecord:
    with self.db.get_session() as session:
        row = AlertRuleRecord(**fields)
        session.add(row)
        session.commit()
        session.refresh(row)
        return row
```

### SQLite write serialization with retry

For write-heavy operations on SQLite, the `DatabaseManager._run_write_transaction()` method wraps operations with `BEGIN IMMEDIATE` and automatic retry on lock errors:

```python
# src/storage.py, line 920-961
def _run_write_transaction(
    self, operation_name: str, write_operation: Callable[[Session], T],
) -> T:
    max_retries = self._sqlite_write_retry_max if self._is_sqlite_engine else 0
    for attempt in range(max_retries + 1):
        session = self.get_session()
        try:
            if self._is_sqlite_engine:
                session.connection().exec_driver_sql("BEGIN IMMEDIATE")
            result = write_operation(session)
            session.commit()
            return result
        except OperationalError as exc:
            session.rollback()
            if (
                self._is_sqlite_engine
                and self._is_sqlite_locked_error(exc)
                and attempt < max_retries
            ):
                delay = self._sqlite_write_retry_base_delay * (2 ** attempt)
                # ...log warning...
                if delay > 0:
                    time.sleep(delay)
                continue
            raise
        except Exception:
            session.rollback()
            raise
        finally:
            session.close()
```

### Portfolio write context manager

The `PortfolioRepository` provides a dedicated `portfolio_write_session()` context manager that acquires the SQLite write lock upfront:

```python
# src/repositories/portfolio_repo.py, line 136-159
@contextmanager
def portfolio_write_session(self):
    session = self.db.get_session()
    try:
        session.connection().exec_driver_sql("BEGIN IMMEDIATE")
    except OperationalError as exc:
        session.close()
        if self._is_sqlite_locked_error(exc):
            raise PortfolioBusyError("Portfolio ledger is busy; please retry shortly.") from exc
        raise
    try:
        yield session
        session.commit()
    except OperationalError as exc:
        session.rollback()
        if self._is_sqlite_locked_error(exc):
            raise PortfolioBusyError("Portfolio ledger is busy; please retry shortly.") from exc
        raise
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()
```

---

## IntegrityError / OperationalError Handling

### Lock detection

Detect SQLite lock errors by inspecting the error text:

```python
# src/storage.py, line 963-973
@staticmethod
def _is_sqlite_locked_error(exc: OperationalError) -> bool:
    err_text = str(getattr(exc, "orig", exc)).lower()
    return any(
        token in err_text
        for token in (
            "database is locked",
            "database schema is locked",
            "database table is locked",
        )
    )
```

### Translating integrity errors to domain exceptions

Catch `IntegrityError` and translate to domain-specific exceptions:

```python
# src/repositories/portfolio_repo.py, line 893-914
@staticmethod
def _translate_trade_integrity_error(
    *, exc: IntegrityError, account_id: int,
    trade_uid: Optional[str], dedup_hash: Optional[str],
) -> Exception:
    err_text = str(getattr(exc, "orig", exc)).lower()
    if trade_uid and ("uix_portfolio_trade_uid" in err_text or "unique" in err_text):
        return DuplicateTradeUidError(
            f"Duplicate trade_uid for account_id={account_id}: {trade_uid}"
        )
    if dedup_hash and (
        "uix_portfolio_trade_dedup_hash" in err_text
        or "portfolio_trades.account_id, portfolio_trades.dedup_hash" in err_text
        or ("unique" in err_text and "dedup_hash" in err_text)
    ):
        return DuplicateTradeDedupHashError(
            f"Duplicate dedup_hash for account_id={account_id}: {dedup_hash}"
        )
    return exc
```

### Custom domain exceptions per repository

Each repository defines its own exception classes at the top of the file:

```python
# src/repositories/portfolio_repo.py, line 33-43
class DuplicateTradeUidError(Exception):
    """Raised when trade_uid conflicts with existing record in one account."""

class DuplicateTradeDedupHashError(Exception):
    """Raised when dedup hash conflicts with existing record in one account."""

class PortfolioBusyError(Exception):
    """Raised when SQLite write serialization cannot acquire the ledger lock."""
```

---

## Migrations

This project does not use Alembic or any migration framework. Schema management is done through:

1. **`Base.metadata.create_all(engine)`** on application startup -- creates tables that don't exist
2. **Manual schema changes** -- modify model definitions in `src/storage.py` and restart

Since SQLite does not support `ALTER TABLE` for most column changes, schema evolution typically requires exporting data, recreating tables, and reimporting.

---

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Table names | `snake_case`, plural for entities | `stock_daily`, `portfolio_accounts`, `backtest_results` |
| Column names | `snake_case` | `created_at`, `data_source`, `trade_date` |
| Primary key | Always `id` with autoincrement | `id = Column(Integer, primary_key=True, autoincrement=True)` |
| Foreign keys | `{entity}_id` | `account_id`, `analysis_history_id` |
| Index names | `ix_{table}_{columns}` | `ix_code_date`, `ix_backtest_code_date` |
| Unique constraint names | `uix_{table}_{columns}` | `uix_code_date`, `uix_portfolio_trade_uid` |

---

## Common Mistakes

1. **Using legacy `session.query()` instead of `select()`** -- Always use SQLAlchemy 2.0 style: `session.execute(select(Model).where(...))`
2. **Forgetting to commit writes** -- Always call `session.commit()` after add/update/delete
3. **Not closing sessions** -- Always use `with self.db.get_session() as session:` or explicit `session.close()` in a `finally` block
4. **Not chunking large upserts on SQLite** -- SQLite has a bind-parameter limit (~999). Chunk to 50 records per upsert.
5. **Catching generic Exception instead of IntegrityError/OperationalError** -- Always translate to domain-specific exceptions so callers can handle them appropriately
6. **SQLite WAL mode not enabling by default** -- Ensure `PRAGMA journal_mode=WAL` is configured for write-heavy workloads
