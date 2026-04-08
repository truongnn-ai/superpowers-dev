# Node.js Logging Setup (Express + pino)

## Complete file listing

### package.json additions
```json
{
  "dependencies": {
    "pino": "^9.0.0",
    "pino-http": "^10.0.0",
    "uuid": "^10.0.0"
  }
}
```

### lib/logger.js
```javascript
const pino = require('pino');
const { AsyncLocalStorage } = require('async_hooks');

const als = new AsyncLocalStorage();

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  timestamp: pino.stdTimeFunctions.isoTime,
  formatters: {
    level: (label) => ({ level: label }),
  },
  mixin() {
    const store = als.getStore();
    return {
      service: process.env.SERVICE_NAME || 'unknown',
      trace_id: store?.traceId || 'no-trace',
      span_id: store?.spanId || 'no-span',
    };
  },
  // JSON output only in production / when LOG_FORMAT=json
  ...(process.env.LOG_FORMAT !== 'json' && process.env.NODE_ENV !== 'production'
    ? { transport: { target: 'pino-pretty' } }
    : {}),
});

module.exports = { logger, als };
```

### middleware/tracing.js
```javascript
const { v4: uuid } = require('uuid');
const { logger, als } = require('../lib/logger');

function tracingMiddleware(req, res, next) {
  const traceId = req.headers['x-trace-id'] || uuid();
  const spanId = uuid().slice(0, 8);
  const start = Date.now();

  als.run({ traceId, spanId }, () => {
    logger.info({ method: req.method, path: req.path }, 'request_started');

    res.setHeader('x-trace-id', traceId);

    res.on('finish', () => {
      const duration_ms = Date.now() - start;
      logger.info(
        { method: req.method, path: req.path, status: res.statusCode, duration_ms },
        'request_completed'
      );
    });

    next();
  });
}

module.exports = { tracingMiddleware };
```

### lib/http-client.js (outgoing calls with trace propagation)
```javascript
const { logger, als } = require('./logger');

async function callService(url, options = {}) {
  const store = als.getStore();
  const traceId = store?.traceId || 'no-trace';

  const headers = {
    ...options.headers,
    'x-trace-id': traceId,
    'content-type': 'application/json',
  };

  logger.info({ target: url, method: options.method || 'GET' }, 'outgoing_call_started');
  const start = Date.now();

  try {
    const response = await fetch(url, {
      ...options,
      headers,
      signal: AbortSignal.timeout(options.timeout || 10000),
    });

    const duration_ms = Date.now() - start;
    logger.info({ target: url, status: response.status, duration_ms }, 'outgoing_call_completed');

    if (!response.ok) {
      const body = await response.text().catch(() => '');
      logger.error(
        { error_category: 'connection', error_detail: `${response.status}: ${body.slice(0, 200)}` },
        'outgoing_call_failed'
      );
    }

    return response;
  } catch (err) {
    const duration_ms = Date.now() - start;

    if (err.name === 'TimeoutError' || err.name === 'AbortError') {
      logger.error(
        { error_category: 'timeout', error_detail: `${url} timed out after ${duration_ms}ms` },
        'outgoing_call_timeout'
      );
    } else if (err.cause?.code === 'ECONNREFUSED' || err.cause?.code === 'ECONNRESET') {
      logger.error(
        { error_category: 'connection', error_detail: `Failed to connect to ${url}` },
        'outgoing_call_connection_failed'
      );
    } else {
      logger.error(
        { error_category: 'unknown', error_detail: err.message?.slice(0, 200) },
        'outgoing_call_unexpected_error'
      );
    }

    throw err;
  }
}

module.exports = { callService };
```

### app.js (wiring it together)
```javascript
const express = require('express');
const { logger } = require('./lib/logger');
const { tracingMiddleware } = require('./middleware/tracing');

const app = express();
const PORT = process.env.PORT || 4000;

app.use(express.json());
app.use(tracingMiddleware);

app.get('/health', (req, res) => {
  res.json({ status: 'ok', service: process.env.SERVICE_NAME });
});

app.listen(PORT, () => {
  logger.info(
    { port: PORT, environment: process.env.NODE_ENV || 'development' },
    'service_started'
  );
});

module.exports = app;
```

## Next.js API routes

For Next.js App Router, wrap API route handlers:

```javascript
// lib/with-tracing.js
import { logger, als } from './logger';
import { v4 as uuid } from 'uuid';

export function withTracing(handler) {
  return async (request) => {
    const traceId = request.headers.get('x-trace-id') || uuid();
    const start = Date.now();

    return als.run({ traceId }, async () => {
      logger.info({ method: request.method, path: request.url }, 'request_started');

      try {
        const response = await handler(request);
        const duration_ms = Date.now() - start;
        logger.info({ status: response.status, duration_ms }, 'request_completed');

        response.headers.set('x-trace-id', traceId);
        return response;
      } catch (err) {
        const duration_ms = Date.now() - start;
        logger.error(
          { error_category: 'unknown', error_detail: err.message?.slice(0, 200), duration_ms },
          'request_failed'
        );
        return Response.json({ error: 'Internal error' }, { status: 500 });
      }
    });
  };
}

// app/api/data/route.js
import { withTracing } from '@/lib/with-tracing';

export const GET = withTracing(async (request) => {
  // handler code here
  return Response.json({ data: [] });
});
```
