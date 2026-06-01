# Error Handling

> How errors are handled in this project.

---

## Overview

This project uses a layered error handling strategy:

1. **Domain exceptions** -- Custom exception classes defined in the module where they are raised
2. **Repository-level translation** -- `IntegrityError` from SQLAlchemy is translated to domain exceptions
3. **FastAPI middleware** -- `ErrorHandlerMiddleware` catches unhandled exceptions and returns a standard JSON response
4. **FastAPI exception handlers** -- Specific handlers for `HTTPException`, `RequestValidationError`, and generic `Exception`
5. **LLM error recovery** -- LiteLLM errors are classified and retried with parameter recovery
6. **External call retry** -- `tenacity` library for retry with exponential backoff on transient errors

---

## Standard API Error Response Format

All API errors use a consistent JSON structure:

```json
{
    "error": "string",       // Machine-readable error code
    "message": "string",     // Human-readable message
    "detail": any            // Optional debugging details (null in production)
}
```

---

## Custom Exception Hierarchy

### Pattern: Base exception class with subclasses

Exception hierarchies use a base class (typically inheriting from `ValueError` or `Exception`) with specialized subclasses:

```python
# src/services/alert_service.py, line 79-94
class AlertServiceError(ValueError):
    """Raised when alert service input is invalid."""
    error_code = "validation_error"

class AlertNotFoundError(AlertServiceError):
    """Raised when an alert resource does not exist."""
    error_code = "not_found"

class UnsupportedAlertTypeError(AlertServiceError):
    """Raised when the API receives a future/non-runtime alert type."""
    error_code = "unsupported_alert_type"
```

### All custom exceptions in the codebase

| Exception | Location | Base Class | Purpose |
|-----------|----------|-----------|---------|
| `DuplicateTradeUidError` | `src/repositories/portfolio_repo.py:33` | `Exception` | Duplicate trade_uid on insert |
| `DuplicateTradeDedupHashError` | `src/repositories/portfolio_repo.py:37` | `Exception` | Duplicate dedup_hash on insert |
| `PortfolioBusyError` | `src/repositories/portfolio_repo.py:41` | `Exception` | SQLite write lock contention |
| `AlertServiceError` | `src/services/alert_service.py:79` | `ValueError` | Alert input validation |
| `AlertNotFoundError` | `src/services/alert_service.py:85` | `AlertServiceError` | Alert resource missing |
| `UnsupportedAlertTypeError` | `src/services/alert_service.py:91` | `AlertServiceError` | Unknown alert type |
| `ConfigValidationError` | `src/services/system_config_service.py:54` | `Exception` | Config fails validation |
| `ConfigConflictError` | `src/services/system_config_service.py:62` | `Exception` | Config key conflicts |
| `ConfigImportError` | `src/services/system_config_service.py:70` | `Exception` | Config import failure |
| `DuplicateTaskError` | `src/services/task_queue.py:124` | `Exception` | Duplicate task submission |
| `MarkdownReportGenerationError` | `src/services/history_service.py:46` | `Exception` | Report generation failure |
| `PortfolioConflictError` | `src/services/portfolio_service.py:40` | `Exception` | Portfolio state conflict |

### Exception design rules

- Custom exceptions are defined in the **same file** where they are raised (not in a central exceptions module)
- Exception class names end with `Error`
- Exceptions carry an `error_code` class attribute when used in the API layer
- Subclass from `ValueError` for validation errors, `Exception` for everything else

---

## FastAPI Middleware Approach

### ErrorHandlerMiddleware (catch-all)

The global middleware catches all unhandled exceptions and returns a 500 response:

```python
# api/middlewares/error_handler.py, line 24-67
class ErrorHandlerMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next: Callable) -> Response:
        try:
            response = await call_next(request)
            return response
        except Exception as e:
            logger.error(
                f"未处理的异常: {e}\n"
                f"请求路径: {request.url.path}\n"
                f"请求方法: {request.method}\n"
                f"堆栈: {traceback.format_exc()}"
            )
            return JSONResponse(
                status_code=500,
                content={
                    "error": "internal_error",
                    "message": "服务器内部错误，请稍后重试",
                    "detail": str(e) if logger.isEnabledFor(logging.DEBUG) else None
                }
            )
```

Key behavior:
- Logs the full stack trace
- Returns Chinese message to the user
- **Debug vs production**: `detail` field includes `str(e)` only when DEBUG logging is enabled; otherwise `null`
- Error code is always `"internal_error"` for the catch-all handler

### Dedicated exception handlers

The `add_error_handlers()` function registers three handlers on the FastAPI app:

```python
# api/middlewares/error_handler.py, line 70-128
def add_error_handlers(app) -> None:
    @app.exception_handler(HTTPException)
    async def http_exception_handler(request: Request, exc: HTTPException):
        # If detail is already a formatted ErrorResponse dict, use it directly
        if isinstance(exc.detail, dict) and "error" in exc.detail and "message" in exc.detail:
            return JSONResponse(status_code=exc.status_code, content=exc.detail)
        # Otherwise wrap it
        return JSONResponse(status_code=exc.status_code, content={
            "error": "http_error",
            "message": str(exc.detail) if exc.detail else "HTTP Error",
            "detail": None
        })

    @app.exception_handler(RequestValidationError)
    async def validation_exception_handler(request: Request, exc: RequestValidationError):
        return JSONResponse(status_code=422, content={
            "error": "validation_error",
            "message": "请求参数验证失败",
            "detail": exc.errors()
        })

    @app.exception_handler(Exception)
    async def general_exception_handler(request: Request, exc: Exception):
        logger.error(f"未处理的异常: {exc}\n请求路径: {request.url.path}\n堆栈: {traceback.format_exc()}")
        return JSONResponse(status_code=500, content={
            "error": "internal_error",
            "message": "服务器内部错误",
            "detail": None
        })
```

Key behavior:
- `HTTPException`: Preserves status code and detail. If detail is already a valid error dict, it is passed through.
- `RequestValidationError`: Returns 422 with `"validation_error"` code and the actual validation errors in `detail`
- `Exception` (fallback): Returns 500 with `"internal_error"`, no detail (production-safe)

---

## LiteLLM Error Classification and Parameter Recovery

For LLM API calls, the codebase implements intelligent error classification and one-shot parameter recovery:

```python
# src/llm/errors.py, line 80-112
def classify_litellm_generation_param_error(
    error: BaseException,
) -> Optional[GenerationParamRecovery]:
    """Classify explicit provider parameter errors into a safe one-shot recovery."""
    text = _normalized_error_text(error)
    if not text:
        return None

    if "temperature" in text:
        allowed_temperature = _parse_allowed_temperature(text)
        if allowed_temperature is not None:
            return GenerationParamRecovery(
                set_params={"temperature": allowed_temperature},
                reason="temperature_default_only",
            )
        if "only" in text and "default" in text:
            return GenerationParamRecovery(
                omit_params=("temperature",),
                reason="temperature_default_only",
            )
        if any(marker in text for marker in _UNSUPPORTED_PARAM_MARKERS):
            return GenerationParamRecovery(
                omit_params=("temperature",),
                reason="temperature_unsupported",
            )

    for param in ("top_p", "presence_penalty", "frequency_penalty", "seed"):
        if param in text and any(marker in text for marker in _UNSUPPORTED_PARAM_MARKERS):
            return GenerationParamRecovery(
                omit_params=(param,),
                reason=f"{param}_unsupported",
            )
    return None
```

The `call_litellm_with_param_recovery()` function (line 115-151) wraps LiteLLM calls: on error, it classifies the error, applies recovery (omit or set params), and retries once:

```python
# src/llm/errors.py, line 127-151
try:
    return call(effective_kwargs)
except Exception as exc:
    recovery = classify_litellm_generation_param_error(exc)
    if recovery is None:
        raise
    retry_kwargs = apply_litellm_param_recovery(effective_kwargs, recovery)
    if retry_kwargs == effective_kwargs:
        raise
    if logger is not None:
        logger.warning(
            "%s %s generation parameter rejected (%s), retrying once with request-scoped recovery",
            log_label, model, recovery.reason,
        )
    response = call(retry_kwargs)
    if cache_recovery:
        remember_litellm_generation_param_recovery(model, recovery, ...)
    return response
```

The recovery is **cached per model** so subsequent calls skip the failed parameter entirely.

---

## Tenacity for Retry

External calls (data fetchers, HTTP requests) use `tenacity` for retry with exponential backoff:

```python
# src/services/social_sentiment_service.py, line 42-47
@retry(
    stop=stop_after_attempt(2),
    wait=wait_exponential(multiplier=1, min=1, max=5),
    retry=retry_if_exception_type(_TRANSIENT_EXCEPTIONS),
    before_sleep=before_sleep_log(logger, logging.WARNING),
    reraise=True,
)
```

Key patterns:
- `stop_after_attempt(2)` -- typically 2-3 attempts for fetchers
- `wait_exponential(multiplier=1, min=1, max=5)` -- exponential backoff capped at 5 seconds
- `retry_if_exception_type(...)` -- only retry on transient errors (SSLError, ConnectionError, Timeout)
- `before_sleep=before_sleep_log(logger, logging.WARNING)` -- log before each retry
- `reraise=True` -- re-raise the original exception after exhausting retries

---

## Error Handling Patterns

### Repository: translate DB errors to domain errors

```python
# src/repositories/portfolio_repo.py, line 343-352
try:
    session.flush()
except IntegrityError as exc:
    raise self._translate_trade_integrity_error(
        exc=exc, account_id=account_id, trade_uid=trade_uid, dedup_hash=dedup_hash,
    ) from exc
```

### Service: raise domain exception on validation failure

```python
# src/services/alert_service.py, line 108-111
def get_rule(self, rule_id: int) -> Dict[str, Any]:
    row = self.repo.get_rule(rule_id)
    if row is None:
        raise AlertNotFoundError(f"Alert rule not found: {rule_id}")
    return self._serialize_rule(row)
```

### API: raise HTTPException for client errors

```python
from fastapi import HTTPException

raise HTTPException(
    status_code=404,
    detail={"error": "not_found", "message": f"Alert rule not found: {rule_id}"}
)
```

Note: When `detail` is a dict with `"error"` and `"message"` keys, the `http_exception_handler` passes it through without modification.

---

## Common Mistakes

1. **Raising generic `Exception` instead of a domain-specific class** -- Always use the appropriate domain exception so callers can handle it specifically
2. **Letting SQLAlchemy errors propagate to the API layer** -- Always translate `IntegrityError` / `OperationalError` to domain exceptions in the repository
3. **Exposing internal error details in production** -- The middleware pattern handles this: `detail` is `None` in the fallback handler
4. **Not logging enough context on errors** -- Always include at minimum the request path, method, and exception traceback
5. **Silently swallowing exceptions without logging** -- Always log at `logger.error()` level before returning an error response
