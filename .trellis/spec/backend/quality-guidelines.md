# Quality Guidelines

> Code quality standards for backend development.

---

## Overview

This project enforces code quality through automated tooling configured in `pyproject.toml` and `setup.cfg`. All code must pass formatting, linting, import sorting, and security checks before being merged.

---

## Code Formatting

### black

All Python code is formatted with **black** at `line-length=120`:

```ini
# pyproject.toml, line 1-15
[tool.black]
line-length = 120
target-version = ['py310', 'py311', 'py312']
include = '\.pyi?$'
exclude = '''
/(
    \.git
    | \.hg
    | \.mypy_cache
    | \.tox
    | \.venv
    | venv
    | _build
    | buck-out
    | build
    | dist
    | __pycache__
)/
'''
```

### isort

Import sorting uses **isort** with the `black` profile:

```ini
# pyproject.toml, line 21-24 / setup.cfg, line 44-49
[tool.isort]
profile = "black"
line_length = 120
skip = [".git", "__pycache__", ".env", "venv", ".venv"]
```

```ini
# setup.cfg, line 44-49
[isort]
profile = black
line_length = 120
skip = .git,__pycache__,.env,venv,.venv,local,node_modules
skip_glob = */node_modules/*
known_first_party = config,storage,analyzer,notification,scheduler,search_service,market_analyzer,stock_analyzer,data_provider
```

### flake8

Linting uses **flake8** with specific rules suppressed to avoid conflicts with black:

```ini
# setup.cfg, line 1-20
[flake8]
max-line-length = 120
exclude =
    .git,
    __pycache__,
    .env,
    venv,
    .venv,
    .venv*,
    build,
    dist,
    local,
    node_modules,
    */node_modules/*,
    *.egg-info
# E501: line too long (some places legitimately need long lines)
# W503: operator before line break (conflicts with black)
# E203: space before slice colon (conflicts with black)
# E402: module-level import not at top of file (sometimes need to set env vars first)
ignore = E501,W503,E203,E402
```

The four suppressed rules:
- **E501** (line too long): Trust black to handle line wrapping at 120 chars
- **W503** (operator before line break): black puts operators at line start
- **E203** (whitespace before `:`): black uses whitespace around slice colons
- **E402** (import not at top): Some modules need to set environment variables before importing

---

## Testing Standards

### pytest

Tests are discovered and run with **pytest**:

```ini
# setup.cfg, line 22-42
[tool:pytest]
testpaths = .
python_files = test_*.py
python_functions = test_*
addopts = -v --tb=short
norecursedirs =
    .git
    __pycache__
    .env
    venv
    .venv
    .venv*
    build
    dist
    local
    node_modules
    */node_modules/*
markers =
    unit: fast offline unit tests
    integration: service-level integration tests without external network dependency
    network: tests requiring external network or third-party services
```

### Test file conventions

- Test files are named `test_<module>.py` (e.g., `test_alert_api.py`, `test_alert_worker.py`)
- Test functions are named `test_<behavior>` or `test_<scenario>`
- Shared fixtures live in `tests/conftest.py` (currently provides asyncio compatibility patches and a `_ThreadlessTestClient` replacement)

### Test markers

Three markers categorize tests by their external dependency level:

| Marker | Description | Runtime |
|--------|-------------|---------|
| `unit` | Fast offline unit tests | Milliseconds |
| `integration` | Service-level tests without external network | Seconds |
| `network` | Tests requiring external network or third-party services | Variable (may be slow or flaky) |

Run a specific category:
```bash
pytest -m unit          # Only unit tests
pytest -m "not network" # Skip network tests
```

### Test setup/teardown pattern

Most tests use `unittest.TestCase` with `setUp`/`tearDown` to reset singletons between tests. The standard pattern resets both `Config` and `DatabaseManager`:

```python
# tests/test_alert_api.py, line 63-73 (representative pattern)
import unittest
from src.config import Config
from src.storage import DatabaseManager

class TestAlertAPI(unittest.TestCase):
    def setUp(self):
        # ... create temp directory, write .env, set environment ...
        Config.reset_instance()
        DatabaseManager.reset_instance()
        self.app = create_app()
        self.client = TestClient(self.app)

    def tearDown(self):
        DatabaseManager.reset_instance()
        Config.reset_instance()
        # ... clean up temp files and environment ...
```

---

## Security Scanning

### bandit

Security linting uses **bandit** with assertions allowed in test files:

```ini
# pyproject.toml, line 26-28
[tool.bandit]
exclude_dirs = ["tests", "test_*.py"]
skips = ["B101"]  # assert statements are allowed in tests
```

- `B101` (assert usage): Skipped because `assert` is the standard pattern in pytest
- Test directories are excluded from bandit scanning

---

## Type Hints

### `from __future__ import annotations`

The project uses PEP 563 lazy evaluation of annotations via a file-level import:

```python
from __future__ import annotations
```

This is used in virtually every non-trivial source file (over 30+ files use it):

```python
# src/llm/errors.py, line 4
from __future__ import annotations

# src/repositories/portfolio_repo.py, line 7
from __future__ import annotations

# src/services/history_service.py, line 12
from __future__ import annotations

# main.py, line 24
from __future__ import annotations
```

Benefits:
- Avoids circular import issues with type hints
- Enables forward references without string quoting
- All annotation types are strings at runtime, reducing import cost

### Type hint conventions

- Use `Optional[X]` for nullable values (not `X | None` for backward compatibility)
- Use `List[X]`, `Dict[K, V]`, `Tuple[X, Y]` from `typing` (not built-in generics `list[X]`)
- Use `Any` sparingly -- only for truly dynamic values or third-party library interfaces
- Use `TYPE_CHECKING` for imports needed only for type annotations:

```python
# src/storage.py, line 64-65
if TYPE_CHECKING:
    from src.search_service import SearchResponse
```

---

## Docstrings and Comments

### Language: Chinese for documentation

The project uses **Chinese** for module-level docstrings, class docstrings, and function docstrings. Code identifiers (variable names, function names, class names) remain in English.

```python
# src/storage.py, lines 1-12
"""
===================================
A股自选股智能分析系统 - 存储层
===================================

职责：
1. 管理 SQLite 数据库连接（单例模式）
2. 定义 ORM 数据模型
3. 提供数据存取接口
4. 实现智能更新逻辑（断点续传）
"""
```

```python
# src/logging_config.py, lines 1-11
"""
===================================
日志配置模块 - 统一的日志系统初始化
===================================

职责：
1. 提供统一的日志格式和配置常量
2. 支持控制台 + 文件（常规/调试）三层日志输出
3. 自动降低第三方库日志级别
"""
```

### Docstring format

- Module-level: title + responsibilities list in Chinese
- Class-level: one-line summary in Chinese, optionally followed by more detail
- Method-level: standard `Args:` / `Returns:` / `Raises:` sections (labels in English, descriptions in Chinese)
- Every method has a docstring, even if brief

```python
# src/storage.py, line 1668-1689 (method docstring example)
def save_daily_data(
    self, df: pd.DataFrame, code: str, data_source: str = "Unknown"
) -> int:
    """
    保存日线数据到数据库

    策略：
    - 按 `(code, date)` 做批量 UPSERT，已存在记录会覆盖更新
    - 同一批次内若存在重复日期，以最后一条记录为准
    - SQLite 分支按 chunk 写入以避免绑定参数上限

    Args:
        df: 包含日线数据的 DataFrame
        code: 股票代码
        data_source: 数据来源名称

    Returns:
        本次实际新增的记录数（不含更新）
    """
```

---

## Required Patterns

1. **`# -*- coding: utf-8 -*-`** on the first line of every `.py` file
2. **`from __future__ import annotations`** on line 2-4 of every file that uses type hints
3. **`logger = logging.getLogger(__name__)`** at module level after imports
4. **Repository pattern** for all database access: one repository class per domain
5. **Service pattern** for all business logic: one service class per domain
6. **Constructor injection** for dependencies: `__init__(self, db_manager: Optional[DatabaseManager] = None)`
7. **Domain-specific exceptions**: custom exception classes in the same file where they are raised
8. **Property-based boolean checks** on SQLAlchemy models: `Model.field.is_(True)` not `Model.field == True`

---

## Forbidden Patterns

1. **`print()` for logging** -- Always use the `logging` module
2. **`session.query()` (legacy SQLAlchemy)** -- Always use `select()` from `sqlalchemy`
3. **String-interpolated SQL** -- Always use parameterized queries via SQLAlchemy ORM
4. **Bare `except:` without specifying exception type** -- Always catch specific exceptions
5. **Swallowing exceptions silently** -- At minimum, log the exception before suppressing it
6. **Circular imports** -- Use `TYPE_CHECKING` and `from __future__ import annotations` to avoid them
7. **Hardcoded paths or credentials** -- Use `src.config.get_config()` or environment variables
8. **Monkey-patching without a clear comment** -- If patches are needed (as in `src/patches/`), document why

---

## Code Review Checklist

When reviewing code, verify:

- [ ] All new files have `# -*- coding: utf-8 -*-` header
- [ ] All new files have appropriate module-level docstring in Chinese
- [ ] `from __future__ import annotations` is present if type hints are used
- [ ] `logger = logging.getLogger(__name__)` is used instead of `print()`
- [ ] SQLAlchemy queries use `select()` syntax (not `session.query()`)
- [ ] Database access goes through a Repository class
- [ ] Business logic lives in a Service class, not in route handlers
- [ ] Custom exceptions are defined for domain errors (not generic `Exception`)
- [ ] `IntegrityError` and `OperationalError` are caught and translated in repositories
- [ ] API error responses follow the `{"error": ..., "message": ..., "detail": ...}` format
- [ ] New features have corresponding unit/integration tests
- [ ] Type hints are present on all public function/method signatures
- [ ] Code passes `black --check`, `isort --check`, and `flake8`
