---
name: structured-logs
description: >
  Instrument generated code with structured JSON logging, correlation IDs, and
  error categorization. Use when: generating a new codebase or service, adding
  logging to existing generated code, setting up observability, preparing code
  for DST simulation or autonomous verification, or when the user says "add logging",
  "make it observable", "add tracing", "instrument the code", "structured logs",
  or "I need to debug this in production". Also use automatically whenever
  generating backend services, API handlers, database clients, or any code that
  makes network calls — these MUST have structured logging from day one.
---

# Structured Logs Skill

Ensure every generated service emits structured, machine-parseable JSON logs that
feed into the autonomous verification pipeline (DST, verification-gate, alerting).

## Why this matters

In an autonomous code generation pipeline, logs are not for humans — they are
**machine-readable verification signals**. The DST simulation skill parses these
logs to correlate faults with errors. The verification-gate reads `error_category`
to route fixes to the right agent. Without structured logs, the feedback loop is blind.

## Core contract

Every log line emitted by generated code MUST be a single JSON object on stdout
with these mandatory fields:

```json
{
  "timestamp": "2026-03-30T12:00:00.000Z",
  "level": "info",
  "service": "backend",
  "message": "Human-readable description of what happened",
  "trace_id": "req-abc-123",
  "span_id": "span-456",
  "context": {}
}
```

Error-level logs MUST additionally include:

```json
{
  "error_category": "timeout|connection|validation|logic|auth|rate_limit|unknown",
  "error_detail": "Specific technical detail",
  "stack_trace": "only for unexpected errors, truncated to 10 lines max"
}
```

---

## Error categories (fixed vocabulary)

The generating agent MUST use exactly these categories — they map to fix actions
in the feedback loop:

| Category | When to use | Fix action |
|---|---|---|
| `timeout` | Request/query exceeded time limit | Add timeout config, circuit breaker |
| `connection` | Failed to connect to upstream service | Check URL, retry logic, health check |
| `validation` | Input/output didn't match expected schema | Fix validation rules or input sanitization |
| `logic` | Business rule violation, unexpected state | Fix algorithm or conditional logic |
| `auth` | Authentication or authorization failure | Fix token handling, permissions |
| `rate_limit` | Rate limit exceeded from upstream | Add backoff, queue, or caching |
| `unknown` | Unhandled exception, catch-all | Investigate — this is a gap in error handling |

NEVER use free-form error categories. NEVER log errors without a category.
The `unknown` category should trigger the agent to add proper error handling.

---

## Correlation ID propagation

Every request entering the system gets a `trace_id`. This ID MUST propagate
through every service hop so logs can be correlated across the full chain.

### Pattern: Middleware → Context → Logger

```
Request arrives → Middleware extracts/generates trace_id
  → Stores in request context (async-local / cls / context)
  → Logger automatically includes trace_id in every log line
  → Outgoing HTTP calls forward trace_id in X-Trace-ID header
```

### Implementation per framework

See reference files for complete implementations:
- `references/python-logging.md` — FastAPI + structlog + contextvars
- `references/node-logging.md` — Express/Next.js + pino + AsyncLocalStorage
- `references/go-logging.md` — Go + zerolog + context.Context

---

## What to log (and what NOT to)

### ALWAYS log (INFO level)
- Request received: method, path, trace_id
- Request completed: status code, duration_ms, trace_id
- Outgoing call started: target service, path, trace_id
- Outgoing call completed: status code, duration_ms, trace_id
- Database query: operation type, table, duration_ms (NOT the query itself)
- Startup: service name, port, environment, version

### ALWAYS log (ERROR level with category)
- Failed outgoing calls (connection, timeout)
- Validation failures (with field name, NOT the invalid value if PII)
- Unhandled exceptions (category: unknown)
- Auth failures (category: auth, NOT the credentials)

### NEVER log
- Passwords, tokens, API keys, session IDs
- Full request/response bodies (PII risk, log bloat)
- SQL queries with parameter values
- Stack traces longer than 10 lines

---

## Logging library setup per language

### Python (structlog) — recommended

```python
# app/logging_setup.py
import structlog
import logging
import os
from contextvars import ContextVar

trace_id_var: ContextVar[str] = ContextVar("trace_id", default="no-trace")

def add_trace_id(logger, method_name, event_dict):
    event_dict["trace_id"] = trace_id_var.get()
    return event_dict

structlog.configure(
    processors=[
        add_trace_id,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer()
    ],
    wrapper_class=structlog.make_filtering_bound_logger(
        logging.getLevelName(os.getenv("LOG_LEVEL", "INFO"))
    ),
)

log = structlog.get_logger()
```

FastAPI middleware:
```python
# app/middleware/tracing.py
import uuid
from starlette.middleware.base import BaseHTTPMiddleware
from app.logging_setup import trace_id_var, log

class TracingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        tid = request.headers.get("x-trace-id", str(uuid.uuid4()))
        trace_id_var.set(tid)
        log.info("request_started", service="backend",
                 method=request.method, path=request.url.path)
        response = await call_next(request)
        log.info("request_completed", service="backend",
                 method=request.method, path=request.url.path,
                 status=response.status_code)
        response.headers["x-trace-id"] = tid
        return response
```

### Node.js (pino) — recommended

```javascript
// lib/logger.js
const pino = require('pino');
const { AsyncLocalStorage } = require('async_hooks');

const als = new AsyncLocalStorage();

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  timestamp: pino.stdTimeFunctions.isoTime,
  formatters: { level: (label) => ({ level: label }) },
  mixin() {
    const store = als.getStore();
    return { service: process.env.SERVICE_NAME, trace_id: store?.traceId || 'no-trace' };
  }
});

module.exports = { logger, als };
```

Express middleware:
```javascript
// middleware/tracing.js
const { v4: uuid } = require('uuid');
const { logger, als } = require('../lib/logger');

function tracingMiddleware(req, res, next) {
  const traceId = req.headers['x-trace-id'] || uuid();
  als.run({ traceId }, () => {
    logger.info({ method: req.method, path: req.path }, 'request_started');
    res.on('finish', () => {
      logger.info({ method: req.method, path: req.path, status: res.statusCode }, 'request_completed');
    });
    res.setHeader('x-trace-id', traceId);
    next();
  });
}
```

### Go (zerolog) — recommended

```go
// pkg/logging/logger.go
package logging

import (
    "context"
    "os"
    "github.com/rs/zerolog"
)

type ctxKey struct{}

func Init() zerolog.Logger {
    return zerolog.New(os.Stdout).With().
        Timestamp().
        Str("service", os.Getenv("SERVICE_NAME")).
        Logger()
}

func WithTraceID(ctx context.Context, traceID string) context.Context {
    return context.WithValue(ctx, ctxKey{}, traceID)
}

func FromCtx(ctx context.Context, logger zerolog.Logger) zerolog.Logger {
    if tid, ok := ctx.Value(ctxKey{}).(string); ok {
        return logger.With().Str("trace_id", tid).Logger()
    }
    return logger
}
```

---

## Error logging pattern

Every `catch`/`except`/`recover` block in generated code MUST follow this pattern:

```python
# Python
try:
    result = await http_client.get(url, timeout=5)
except httpx.TimeoutException as e:
    log.error("upstream_timeout", error_category="timeout",
              error_detail=f"GET {url} timed out after 5s",
              service="backend")
    raise HTTPException(status_code=504, detail="Upstream timeout")
except httpx.ConnectError as e:
    log.error("upstream_connection_failed", error_category="connection",
              error_detail=f"Failed to connect to {url}",
              service="backend")
    raise HTTPException(status_code=502, detail="Upstream unavailable")
except Exception as e:
    log.error("unexpected_error", error_category="unknown",
              error_detail=str(e)[:200],
              service="backend")
    raise HTTPException(status_code=500, detail="Internal error")
```

```javascript
// Node.js
try {
  const res = await fetch(url, { signal: AbortSignal.timeout(5000) });
  return await res.json();
} catch (err) {
  if (err.name === 'TimeoutError') {
    logger.error({ error_category: 'timeout', error_detail: `GET ${url} timed out` }, 'upstream_timeout');
    throw new AppError(504, 'Upstream timeout');
  } else if (err.cause?.code === 'ECONNREFUSED') {
    logger.error({ error_category: 'connection', error_detail: `Failed to connect to ${url}` }, 'upstream_connection_failed');
    throw new AppError(502, 'Upstream unavailable');
  } else {
    logger.error({ error_category: 'unknown', error_detail: err.message?.slice(0, 200) }, 'unexpected_error');
    throw new AppError(500, 'Internal error');
  }
}
```

---

## Docker environment variables

Every generated service's docker-compose entry MUST include:

```yaml
environment:
  LOG_FORMAT: json
  LOG_LEVEL: ${LOG_LEVEL:-info}
  SERVICE_NAME: <service-name>
```

The `LOG_FORMAT: json` flag tells the logging library to output JSON instead of
human-readable format. This is critical — pretty-printed logs break DST log parsing.

---

## Verification checklist

Before considering logging complete, verify:

- [ ] Every service outputs JSON to stdout (no `print()`, no `console.log()` without pino)
- [ ] Every log line has `timestamp`, `level`, `service`, `message`
- [ ] Every error log has `error_category` from the fixed vocabulary
- [ ] `trace_id` propagates across all service-to-service calls via `X-Trace-ID` header
- [ ] No PII in logs (passwords, tokens, emails, request bodies)
- [ ] Docker env vars include `LOG_FORMAT`, `LOG_LEVEL`, `SERVICE_NAME`
- [ ] Startup log emitted on service boot (confirms service identity and port)

---

## Reference files

- `references/python-logging.md` — Complete FastAPI + structlog setup with middleware
- `references/node-logging.md` — Complete Express/Next.js + pino setup with ALS
- `references/go-logging.md` — Complete Go + zerolog setup with context propagation
