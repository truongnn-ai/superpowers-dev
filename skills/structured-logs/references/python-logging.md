# Python Logging Setup (FastAPI + structlog)

## Complete file listing

### requirements.txt additions
```
structlog>=24.1.0
```

### app/logging_setup.py
```python
import structlog
import logging
import os
from contextvars import ContextVar

trace_id_var: ContextVar[str] = ContextVar("trace_id", default="no-trace")
span_id_var: ContextVar[str] = ContextVar("span_id", default="no-span")

def add_trace_context(logger, method_name, event_dict):
    event_dict["trace_id"] = trace_id_var.get()
    event_dict["span_id"] = span_id_var.get()
    return event_dict

def setup_logging():
    log_format = os.getenv("LOG_FORMAT", "json")
    log_level = os.getenv("LOG_LEVEL", "INFO").upper()

    processors = [
        add_trace_context,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
    ]

    if log_format == "json":
        processors.append(structlog.processors.JSONRenderer())
    else:
        processors.append(structlog.dev.ConsoleRenderer())

    structlog.configure(
        processors=processors,
        wrapper_class=structlog.make_filtering_bound_logger(
            logging.getLevelName(log_level)
        ),
    )

    # Suppress noisy third-party loggers
    logging.getLogger("uvicorn.access").setLevel(logging.WARNING)
    logging.getLogger("httpcore").setLevel(logging.WARNING)

setup_logging()
log = structlog.get_logger()
```

### app/middleware/tracing.py
```python
import uuid
import time
from starlette.middleware.base import BaseHTTPMiddleware
from app.logging_setup import trace_id_var, span_id_var, log

class TracingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        trace_id = request.headers.get("x-trace-id", str(uuid.uuid4()))
        span_id = str(uuid.uuid4())[:8]
        trace_id_var.set(trace_id)
        span_id_var.set(span_id)

        start = time.monotonic()
        log.info("request_started",
                 service=request.app.state.service_name,
                 method=request.method,
                 path=request.url.path)

        try:
            response = await call_next(request)
        except Exception as exc:
            duration_ms = round((time.monotonic() - start) * 1000)
            log.error("request_failed",
                      service=request.app.state.service_name,
                      method=request.method,
                      path=request.url.path,
                      duration_ms=duration_ms,
                      error_category="unknown",
                      error_detail=str(exc)[:200])
            raise

        duration_ms = round((time.monotonic() - start) * 1000)
        log.info("request_completed",
                 service=request.app.state.service_name,
                 method=request.method,
                 path=request.url.path,
                 status=response.status_code,
                 duration_ms=duration_ms)

        response.headers["x-trace-id"] = trace_id
        return response
```

### app/http_client.py (outgoing calls with trace propagation)
```python
import httpx
from app.logging_setup import trace_id_var, log

async def call_service(method: str, url: str, **kwargs) -> httpx.Response:
    trace_id = trace_id_var.get()
    headers = kwargs.pop("headers", {})
    headers["x-trace-id"] = trace_id

    log.info("outgoing_call_started", target=url, method=method)

    try:
        async with httpx.AsyncClient(timeout=10.0) as client:
            response = await client.request(method, url, headers=headers, **kwargs)

        log.info("outgoing_call_completed",
                 target=url, status=response.status_code)
        return response

    except httpx.TimeoutException:
        log.error("outgoing_call_timeout",
                  error_category="timeout",
                  error_detail=f"{method} {url} timed out after 10s")
        raise
    except httpx.ConnectError:
        log.error("outgoing_call_connection_failed",
                  error_category="connection",
                  error_detail=f"Failed to connect to {url}")
        raise
```

### app/main.py (wiring it together)
```python
import os
from fastapi import FastAPI
from app.middleware.tracing import TracingMiddleware
from app.logging_setup import log

app = FastAPI()
app.state.service_name = os.getenv("SERVICE_NAME", "backend")
app.add_middleware(TracingMiddleware)

@app.on_event("startup")
async def startup():
    log.info("service_started",
             service=app.state.service_name,
             port=os.getenv("PORT", "8000"),
             environment=os.getenv("ENVIRONMENT", "development"))

@app.get("/health")
async def health():
    return {"status": "ok", "service": app.state.service_name}
```
