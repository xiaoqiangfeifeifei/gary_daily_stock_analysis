# Logging Guidelines

> How logging is done in this project.

---

## Overview

This project uses Python's standard `logging` module configured through a centralized `setup_logging()` function in `src/logging_config.py`.

All logging is initialized once at application startup by calling `setup_logging()` with the desired configuration. After setup, every module obtains its logger via:

```python
logger = logging.getLogger(__name__)
```

---

## Log Format

The standard log format is defined as a constant in `src/logging_config.py`:

```python
# src/logging_config.py, line 22
LOG_FORMAT = "%(asctime)s | %(levelname)-8s | %(pathname)s:%(lineno)d | %(message)s"
LOG_DATE_FORMAT = "%Y-%m-%d %H:%M:%S"
```

Example output:
```
2026-06-01 14:30:15 | INFO     | src/storage.py:848 | 数据库初始化完成: sqlite:///./data/db.sqlite3
2026-06-01 14:30:16 | DEBUG    | src/services/alert_service.py:120 | Evaluating alert rule 42 for AAPL
2026-06-01 14:30:17 | WARNING  | src/llm/errors.py:138 | [LiteLLM] gpt-4o generation parameter rejected (temperature_default_only), retrying once with request-scoped recovery
```

### RelativePathFormatter

The project uses a custom `RelativePathFormatter` that converts absolute file paths to relative paths (from the project root) in log output:

```python
# src/logging_config.py, line 34-48
class RelativePathFormatter(logging.Formatter):
    """Custom Formatter that outputs relative paths instead of absolute paths."""

    def __init__(self, fmt=None, datefmt=None, relative_to=None):
        super().__init__(fmt, datefmt)
        self.relative_to = Path(relative_to) if relative_to else Path.cwd()

    def format(self, record):
        try:
            record.pathname = str(Path(record.pathname).relative_to(self.relative_to))
        except ValueError:
            pass  # Keep absolute path if not relative to project root
        return super().format(record)
```

This makes log output more readable and portable across machines.

---

## Three-Layer Log Output

The `setup_logging()` function configures three simultaneous outputs:

```python
# src/logging_config.py, line 83-89
def setup_logging(
    log_prefix: str = "app",
    log_dir: str = "./logs",
    console_level: Optional[int] = None,
    debug: bool = False,
    extra_quiet_loggers: Optional[List[str]] = None,
) -> None:
```

| Layer | Target | Level | Rotation | Purpose |
|-------|--------|-------|----------|---------|
| 1. Console | `sys.stdout` | INFO (DEBUG if `debug=True`) | None | Development / real-time monitoring |
| 2. Regular file | `logs/<prefix>_<YYYYMMDD>.log` | INFO | 10 MB, 5 backups | Production operation logs |
| 3. Debug file | `logs/<prefix>_debug_<YYYYMMDD>.log` | DEBUG | 50 MB, 3 backups | Detailed debug logs for troubleshooting |

```python
# src/logging_config.py, line 133-158
# Handler 1: Console output
console_handler = logging.StreamHandler(sys.stdout)
console_handler.setLevel(level)
console_handler.setFormatter(rel_formatter)
root_logger.addHandler(console_handler)

# Handler 2: Regular log file (INFO level, 10MB rotation)
file_handler = RotatingFileHandler(log_file, maxBytes=10 * 1024 * 1024, backupCount=5, encoding='utf-8')
file_handler.setLevel(logging.INFO)
file_handler.setFormatter(rel_formatter)
root_logger.addHandler(file_handler)

# Handler 3: Debug log file (DEBUG level, 50MB rotation)
debug_handler = RotatingFileHandler(debug_log_file, maxBytes=50 * 1024 * 1024, backupCount=3, encoding='utf-8')
debug_handler.setLevel(logging.DEBUG)
debug_handler.setFormatter(rel_formatter)
root_logger.addHandler(debug_handler)
```

The root logger is always set to `DEBUG` level -- individual handlers control the output level.

---

## Module-Level Logger Pattern

Every module obtains its logger at module level (not inside functions or classes):

```python
# src/storage.py, line 58
logger = logging.getLogger(__name__)

# src/logging_config.py, line 13 (the config module itself uses the pattern too)
# (Implicitly: logging.info(...) is called within setup_logging)

# src/llm/errors.py -- logger obtained at call-site via parameter injection or via logging.getLogger(__name__)
```

```python
# src/repositories/portfolio_repo.py, line 30
logger = logging.getLogger(__name__)

# src/services/alert_service.py, line 76
logger = logging.getLogger(__name__)

# api/middlewares/error_handler.py, line 21
logger = logging.getLogger(__name__)
```

This pattern is used in **every source file** that does logging. It ensures the logger name matches the module's import path, making it easy to filter logs by module.

---

## Log Level Usage

| Level | When to use | Example |
|-------|-------------|---------|
| `DEBUG` | Detailed diagnostic info, token-by-token failures, parameter values | `logger.debug("数据库引擎已清理")` |
| `INFO` | Normal lifecycle events, initialization, startup | `logger.info(f"数据库初始化完成: {db_url}")` |
| `WARNING` | Recoverable issues, retries, degraded behavior | `logger.warning("SQLite 写入锁冲突，准备重试...")` |
| `ERROR` | Exceptions that need attention but don't crash the app | `logger.error(f"未处理的异常: {e}\n堆栈: {traceback.format_exc()}")` |

### Examples from the codebase

```python
# src/storage.py, line 848
logger.info(f"数据库初始化完成: {db_url}")

# src/storage.py, line 946-952 -- WARNING for retries
logger.warning(
    "SQLite 写入锁冲突，准备重试: %s (%s/%s, %.2fs)",
    operation_name, attempt + 1, max_retries, delay,
)

# api/middlewares/error_handler.py, line 52-57 -- ERROR with stack trace
logger.error(
    f"未处理的异常: {e}\n"
    f"请求路径: {request.url.path}\n"
    f"请求方法: {request.method}\n"
    f"堆栈: {traceback.format_exc()}"
)

# src/llm/errors.py, line 138 -- WARNING for parameter recovery
logger.warning(
    "%s %s generation parameter rejected (%s), retrying once with request-scoped recovery",
    log_label, model, recovery.reason,
)
```

---

## Third-Party Logger Suppression

To reduce noise, the following third-party loggers are set to `WARNING` level by default:

```python
# src/logging_config.py, line 53-58
DEFAULT_QUIET_LOGGERS = [
    'urllib3',
    'sqlalchemy',
    'google',
    'httpx',
]
```

Additional quiet loggers can be passed via the `extra_quiet_loggers` parameter to `setup_logging()`.

### LiteLLM Log Level

LiteLLM log levels are configurable via the `LITELLM_LOG_LEVEL` environment variable:

```python
# src/logging_config.py, line 30
_DEFAULT_LITELLM_LOG_LEVEL = 'WARNING'

# src/logging_config.py, line 60-64
LITELLM_LOGGERS = [
    'LiteLLM',
    'LiteLLM Router',
    'LiteLLM Proxy',
    'litellm',
]
```

If `LITELLM_LOG_LEVEL` is unset or invalid, it defaults to `WARNING`. Valid values: `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`.

```python
# src/logging_config.py, line 68-80
def _resolve_litellm_log_level(raw_level: Optional[str] = None) -> Tuple[int, Optional[str]]:
    if raw_level is None:
        raw_level = os.getenv('LITELLM_LOG_LEVEL', '')
    normalized = (raw_level or '').strip().upper()
    if not normalized:
        normalized = _DEFAULT_LITELLM_LOG_LEVEL
    level = _ALLOWED_LOG_LEVELS.get(normalized)
    if level is None:
        return _ALLOWED_LOG_LEVELS[_DEFAULT_LITELLM_LOG_LEVEL], raw_level
    return level, None
```

---

## What to Log

- **Application lifecycle**: startup, shutdown, initialization of subsystems
- **Errors**: all exceptions with full traceback and request context
- **Retries**: when an operation is retried (warning level, with attempt count and delay)
- **External API calls**: failures and retries (not full request/response bodies)
- **Configuration**: resolved configuration values at startup (sanitized of secrets)

---

## What NOT to Log

- **API keys, tokens, passwords, secrets** -- Never log these. They cause credential leaks.
- **Personally identifiable information (PII)** -- User names are acceptable in moderation; full identity documents or contact details are not.
- **Full request/response bodies** -- Log summaries or error snippets instead. Full bodies bloat logs and may contain PII.
- **Large binary data** -- Never log binary content. Log metadata (size, type) instead.

---

## Common Mistakes

1. **Using `print()` instead of `logger`** -- Always use the logging module; `print()` bypasses log levels, formatting, and file output
2. **Getting the logger inside a function instead of at module level** -- Module-level `logger = logging.getLogger(__name__)` is the standard pattern
3. **Logging at INFO for routine operations that should be DEBUG** -- INFO should be for meaningful lifecycle events; use DEBUG for detailed tracing
4. **Not including enough context in error logs** -- Always include relevant identifiers (e.g., `account_id`, `rule_id`, request path) in error messages
5. **Forgetting to call `setup_logging()` before other modules** -- Logging must be initialized before any module that logs; otherwise messages are lost
