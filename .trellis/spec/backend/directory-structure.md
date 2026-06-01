# Directory Structure

> How backend code is organized in this project.

---

## Overview

This project follows a layered architecture with clear separation between data access, business logic, API endpoints, and external data providers.

The top-level modules are:

- **`src/`** -- Core source code: business logic, data models, repositories, services, utilities.
- **`api/`** -- FastAPI application: route handlers, request/response schemas, middleware.
- **`data_provider/`** -- External data source adapters (AkShare, eFinance, Tushare, etc.).
- **`strategies/`** -- Trading strategy implementations.
- **`bot/`** -- Chat bot integrations (Discord, Feishu, DingTalk).
- **`apps/`** -- Desktop and web application frontends (dsa-desktop, dsa-web).
- **`tests/`** -- Test files with fixtures.

---

## Directory Layout

```
src/
├── agent/                # AI agent logic (conversation, memory, events)
│   ├── chat_context.py
│   ├── events.py
│   ├── memory.py
│   └── provider_trace.py
├── core/                 # Core domain logic
├── data/                 # Data loaders and mappings
│   ├── stock_mapping.py
│   └── stock_index_loader.py
├── llm/                  # LLM adapter, error classification, generation params
│   ├── errors.py
│   ├── generation_params.py
│   └── ...
├── notification_sender/  # Notification channel adapters
├── patches/              # Monkey-patches for third-party libraries
├── repositories/         # Data access layer (one repo per domain)
│   ├── portfolio_repo.py
│   └── alert_repo.py
├── schemas/              # Pydantic/Python data schemas
├── services/             # Business logic layer (one service per domain)
│   ├── alert_service.py
│   ├── portfolio_service.py
│   ├── history_service.py
│   ├── analysis_service.py
│   ├── backtest_service.py
│   ├── stock_service.py
│   ├── task_service.py
│   └── ...
├── utils/                # Shared utility functions
│   ├── sanitize.py
│   └── data_processing.py
├── config.py             # Application configuration
├── storage.py            # SQLAlchemy models + DatabaseManager singleton
├── logging_config.py     # Logging setup
└── analyzer.py           # Core analysis engine

api/
├── __init__.py
├── app.py                 # FastAPI application factory
├── deps.py                # FastAPI dependency injection
├── v1/
│   ├── __init__.py
│   ├── router.py          # API v1 router aggregation
│   ├── endpoints/         # FastAPI route handlers
│   └── schemas/           # FastAPI request/response Pydantic models
└── middlewares/           # FastAPI middleware (auth, error handler)
    ├── __init__.py
    ├── auth.py
    └── error_handler.py

data_provider/            # External data adapters (one file per provider)
├── akshare_fetcher.py
├── efinance_fetcher.py
├── tushare_fetcher.py
├── yfinance_fetcher.py
├── baostock_fetcher.py
├── pytdx_fetcher.py
├── fundamental_adapter.py
├── yfinance_fundamental_adapter.py
├── base.py                # Base classes for providers
└── ...

tests/
├── conftest.py           # Shared fixtures
├── test_alert_api.py
├── test_alert_worker.py
└── ...
```

---

## Module Organization

### Where to put new code

| Concern | Location | Example |
|---------|----------|---------|
| SQLAlchemy models | `src/storage.py` | `StockDaily`, `PortfolioAccount`, `AlertRuleRecord` |
| Database access (CRUD) | `src/repositories/<domain>_repo.py` | `portfolio_repo.py`, `alert_repo.py` |
| Business logic | `src/services/<domain>_service.py` | `alert_service.py`, `history_service.py` |
| API route handlers | `api/v1/endpoints/<resource>.py` | FastAPI route functions |
| API request/response schemas | `api/v1/schemas/<resource>.py` | Pydantic models for request/response |
| External data fetching | `data_provider/<provider>_fetcher.py` | `akshare_fetcher.py` |
| Cross-cutting utilities | `src/utils/<purpose>.py` | `sanitize.py`, `data_processing.py` |
| LLM interaction | `src/llm/` | `errors.py` (error classification), generation params |
| Custom exceptions | In the domain file where they are raised | `DuplicateTradeUidError` in `portfolio_repo.py` |

### Service + Repository pattern

Each domain has a **Repository** (data access) and a **Service** (business logic):

```
src/repositories/alert_repo.py   →  AlertRepository   (raw DB operations)
src/services/alert_service.py     →  AlertService      (business rules, validation)
```

The Service depends on the Repository via constructor injection. Both receive a `DatabaseManager` instance (or default to the singleton).

### One file per domain

New features should follow the existing pattern:
1. Define SQLAlchemy models in `src/storage.py`
2. Create a repository in `src/repositories/<domain>_repo.py`
3. Create a service in `src/services/<domain>_service.py`
4. Add API routes in `api/v1/endpoints/<domain>.py`
5. Add API request/response schemas in `api/v1/schemas/<domain>.py`

---

## Naming Conventions

### Files and directories

- **`snake_case`** for all file and directory names: `alert_service.py`, `portfolio_repo.py`
- Descriptive names that indicate purpose: `history_service.py` (not `hist.py`)
- Provider fetchers use the pattern `<provider>_fetcher.py`: `akshare_fetcher.py`, `yfinance_fetcher.py`

### Classes

- **PascalCase** for classes: `AlertService`, `PortfolioRepository`, `DatabaseManager`
- Domain exception classes end with `Error`: `DuplicateTradeUidError`, `AlertNotFoundError`
- SQLAlchemy model classes are nouns: `StockDaily`, `PortfolioAccount`, `AnalysisHistory`

### Functions and variables

- **`snake_case`** for functions and variables: `get_account()`, `save_daily_data()`
- Private methods prefixed with `_`: `_normalize_daily_date()`, `_is_sqlite_locked_error()`

### Database tables

- **`snake_case`** for table names, usually plural for entities: `stock_daily`, `portfolio_accounts`, `backtest_results`
- Index names: `ix_<table>_<columns>` (e.g., `ix_code_date`, `ix_backtest_code_date`)
- Unique constraint names: `uix_<table>_<columns>` (e.g., `uix_code_date`, `uix_portfolio_trade_uid`)

---

## Examples

### Well-organized module: `src/services/`

The `src/services/` directory is the best example of module organization. Each file is a self-contained domain service:

```
src/services/
├── alert_service.py          # Alert CRUD + evaluation business logic
├── portfolio_service.py      # Portfolio operations
├── history_service.py        # History query + markdown report generation
├── analysis_service.py       # Stock analysis orchestration
├── backtest_service.py       # Backtest evaluation
├── stock_service.py          # Stock data queries
├── task_service.py           # Task queue management
├── system_config_service.py  # System configuration management
├── alert_indicators.py       # Technical indicator alert helpers
├── alert_worker.py           # Alert evaluation worker
├── portfolio_alerts.py       # Portfolio-specific alert logic
├── market_light_alerts.py    # Market-light alert logic
└── ...
```

Key patterns:
- Each `*_service.py` is a class with `__init__(self, db_manager: Optional[DatabaseManager] = None)`
- Helper modules (e.g., `alert_indicators.py`, `portfolio_alerts.py`) contain pure functions
- Custom exceptions defined at module level, before the service class

### Well-organized module: `src/repositories/`

Repository files follow a clean pattern:

```
src/repositories/
├── portfolio_repo.py    # PortfolioRepository with custom exceptions
├── alert_repo.py        # AlertRepository with clean CRUD
├── analysis_repo.py     # Analysis history CRUD
├── backtest_repo.py     # Backtest data access
├── stock_repo.py        # Stock data access
└── ...
```

Key patterns:
- Constructor accepts optional `DatabaseManager`, defaults to `DatabaseManager.get_instance()`
- Methods use `with self.db.get_session() as session:` for read queries
- Write methods use explicit `BEGIN IMMEDIATE` for SQLite serialization
- Custom domain exceptions defined at the top of the file
