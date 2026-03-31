---
name: dst-simulation
description: >
  Simulate a generated codebase in Docker with fault injection to find integration
  bugs autonomously. Use when: code has been generated and needs to be verified
  under realistic failure conditions, running a local deployment test, checking if
  FE and BE integrate correctly, testing resilience to network faults or service
  crashes, validating a full-stack generated app, or any time you need to prove
  generated code actually works end-to-end before trusting it. Also use after code
  generation completes, after passing unit/property tests, or when the user says
  "verify", "simulate", "test deployment", "does this actually work", "fault test",
  "integration check", or "run DST".
context: fork
---

# DST Simulation Skill

Spin up a generated codebase in Docker, inject faults, capture structured logs,
and produce a verification report — all without human intervention.

## Dependencies on other skills

This skill delegates to two other installed skills:
- **`docker-patterns`** (from `affaan-m/everything-claude-code`) — Use for ALL
  Dockerfile generation and docker-compose best practices (multi-stage builds,
  volume strategies, security hardening, networking, health checks). This skill
  does NOT reinvent Docker knowledge — it adds the fault injection layer on top.
- **`structured-logs`** (custom) — Use to ensure every generated service emits
  JSON logs with `error_category` and `trace_id` before running DST. Without
  structured logs, Phase 5 cannot produce a meaningful report.

## When to use

- After an agent generates a full-stack codebase (FE + BE + DB + 3rd party)
- After unit/property tests pass but before trusting the code
- When you need to prove integration correctness under failure conditions
- As the "middle loop" verification signal in an autonomous feedback pipeline

## Overview

The workflow has 5 phases:

```
Phase 0: Prerequisites → Phase 1: Analyze → Phase 2: Compose → Phase 3: Boot → Phase 4: Fault → Phase 5: Report
```

Each phase produces artifacts the next phase consumes. If any phase fails,
produce a structured error and stop — the calling agent needs actionable feedback,
not a wall of logs.

---

## Phase 0: Prerequisites

Before starting DST, verify the dependent skills have been applied:

1. **Check structured logging**: Every service should already have structured JSON
   logging configured via the `structured-logs` skill. Verify by checking for
   logging library imports (structlog/pino/zerolog) and `LOG_FORMAT` env vars.
   If missing → invoke `/structured-logs` first.

2. **Check Docker setup**: The `docker-patterns` skill should be available. It
   provides the Dockerfile and compose generation patterns this skill builds upon.

---

## Container Reuse Mode

When Docker containers are already running (set up by Task 0 in the plan), skip Phase 2-3 and go straight to fault injection.

**Detection:**
```bash
# Check if services are already running and healthy
docker compose ps --format json | jq '.[].Health'
```

**If all services healthy:**
1. Skip Phase 2 (generate compose) — use existing docker-compose.yml
2. Skip Phase 3 (boot) — containers already running
3. Add Toxiproxy as overlay:
   ```bash
   # Start only the DST overlay services alongside existing ones
   docker compose -f docker-compose.yml -f docker-compose.dst.yml up -d toxiproxy wiremock
   ```
4. Proceed to Phase 4 (fault injection)

**If services not running:**
- Fall back to full Phase 0-3 workflow (existing behavior)

**Cleanup in reuse mode:**
- Only stop Toxiproxy/WireMock overlay services
- Do NOT stop the application services (they belong to the dev environment)

---

## Phase 1: Analyze the codebase

Scan the generated project to understand its topology:

1. Identify all services (look for `Dockerfile`, `package.json`, `requirements.txt`,
   `go.mod`, `Cargo.toml`, or framework markers like `next.config.*`, `manage.py`,
   `main.go`)
2. Identify databases (look for migration files, ORM configs, `.sql` files)
3. Identify 3rd-party dependencies (API calls to external URLs, SDK imports)
4. Map service-to-service communication (imports, env vars with URLs, API routes)

Produce a **topology map**:
```json
{
  "services": [
    {"name": "frontend", "type": "nextjs", "port": 3000, "depends_on": ["backend"]},
    {"name": "backend", "type": "fastapi", "port": 8000, "depends_on": ["db"]},
    {"name": "db", "type": "postgres", "port": 5432, "depends_on": []}
  ],
  "edges": [
    {"from": "frontend", "to": "backend", "protocol": "http", "path": "/api/*"},
    {"from": "backend", "to": "db", "protocol": "tcp", "port": 5432}
  ],
  "external_apis": [
    {"service": "backend", "url": "https://api.stripe.com", "mock_with": "wiremock"}
  ]
}
```

If the codebase doesn't have Dockerfiles, generate them by invoking the
`docker-patterns` skill. It handles multi-stage builds, non-root users, slim
base images, .dockerignore, and security hardening. Do NOT write Dockerfile
guidance inline here — `docker-patterns` is the single source of truth for that.

---

## Phase 2: Generate docker-compose.yml

Build a compose file from the topology map in two layers:

### Layer 1: Base compose (delegate to docker-patterns)
Use the `docker-patterns` skill to generate the base docker-compose.yml with:
- All application services from Phase 1
- Database services with volume mounts and init scripts
- Health checks on every service (`interval`, `timeout`, `retries`)
- Proper networking, volume strategies, and security hardening
- Environment variables including `LOG_FORMAT=json`, `LOG_LEVEL=info`,
  `SERVICE_NAME=<name>` (required by `structured-logs` skill)

The `docker-patterns` skill knows the best practices for compose structure,
volume types, depends_on conditions, .dockerignore, and security. Do NOT
duplicate that knowledge here.

### Layer 2: DST overlay (this skill's addition)
On top of the base compose, add these DST-specific services and rewiring:

**Add Toxiproxy sidecar** for fault injection:
```yaml
toxiproxy:
  image: ghcr.io/shopify/toxiproxy:latest
  ports:
    - "8474:8474"   # Toxiproxy API
    - "8001:8001"   # backend proxy
    - "5433:5433"   # db proxy
    - "8002:8002"   # external API proxy
  volumes:
    - ./toxiproxy.json:/config/toxiproxy.json
  command: ["-host=0.0.0.0", "-config=/config/toxiproxy.json"]
  healthcheck:
    test: ["CMD", "wget", "-q", "--spider", "http://localhost:8474/version"]
    interval: 3s
    timeout: 2s
    retries: 5
  networks: [dst-net]
```

**Add WireMock** for any external API mocks:
```yaml
wiremock:
  image: wiremock/wiremock:latest
  ports: ["8080:8080"]
  volumes:
    - ./wiremock:/home/wiremock
  command: ["--verbose"]
  healthcheck:
    test: ["CMD", "wget", "-q", "--spider", "http://localhost:8080/__admin/mappings"]
    interval: 3s
    timeout: 2s
    retries: 5
  networks: [dst-net]
```

**Rewire service URLs through Toxiproxy proxies**:
```
frontend → toxiproxy:8001 → backend:8000
backend  → toxiproxy:5433 → db:5432
backend  → toxiproxy:8002 → wiremock:8080 (mocked external API)
```

Update service environment variables to point to Toxiproxy proxy ports instead
of direct service ports. For example, the backend's `DATABASE_URL` should use
`toxiproxy:5433` not `db:5432`.

**Generate toxiproxy.json** config that pre-creates proxies — one per edge in
the topology map:
```json
[
  {"name": "backend-proxy", "listen": "0.0.0.0:8001", "upstream": "backend:8000"},
  {"name": "db-proxy", "listen": "0.0.0.0:5433", "upstream": "db:5432"},
  {"name": "external-api-proxy", "listen": "0.0.0.0:8002", "upstream": "wiremock:8080"}
]
```

See `references/toxiproxy-recipes.md` for advanced fault scenarios.
See `references/wiremock-stubs.md` for common 3rd-party API mock patterns.

---

## Phase 3: Boot and health check

1. Run `docker compose up -d --build --wait --wait-timeout 120`
2. Poll health endpoints for every service (max 60 seconds)
3. Run a **smoke test**: one happy-path request through the full chain
   (e.g., `curl frontend:3000/api/health` that touches BE and DB)

If boot fails:
```json
{
  "phase": "boot",
  "status": "FAIL",
  "error": "backend failed health check after 60s",
  "logs": "<last 50 lines of backend container logs>",
  "fix_category": "startup_error",
  "suggested_action": "Check backend Dockerfile CMD and database connection string"
}
```

If smoke test fails, capture the request/response and error logs.

---

## Phase 4: Fault injection scenarios

Run each scenario sequentially. For each: inject fault → run test traffic →
capture result → remove fault → verify recovery.

### Scenario 1: Network latency (FE ↔ BE)
```bash
# Add 2000ms latency to backend proxy
curl -s -X POST http://toxiproxy:8474/proxies/backend-proxy/toxics \
  -d '{"name":"latency","type":"latency","attributes":{"latency":2000}}'
# Run test traffic
curl -w "%{http_code} %{time_total}s" http://frontend:3000/api/data
# Remove toxic
curl -s -X DELETE http://toxiproxy:8474/proxies/backend-proxy/toxics/latency
```
**Pass criteria**: Frontend returns response (may be slow). No 500 errors. No unhandled exceptions in logs.

### Scenario 2: Connection reset (FE ↔ BE)
```bash
curl -s -X POST http://toxiproxy:8474/proxies/backend-proxy/toxics \
  -d '{"name":"reset","type":"reset_peer","attributes":{"timeout":0}}'
```
**Pass criteria**: Frontend shows graceful error message, not a crash. Backend logs the connection error with proper error category.

### Scenario 3: Database timeout (BE ↔ DB)
```bash
curl -s -X POST http://toxiproxy:8474/proxies/db-proxy/toxics \
  -d '{"name":"timeout","type":"timeout","attributes":{"timeout":5000}}'
```
**Pass criteria**: Backend returns 503 or appropriate error. No connection pool exhaustion. No data corruption.

### Scenario 4: External API failure (BE ↔ 3rd party)
```bash
curl -s -X POST http://toxiproxy:8474/proxies/external-api-proxy/toxics \
  -d '{"name":"reset","type":"reset_peer","attributes":{"timeout":0}}'
```
**Pass criteria**: Backend handles missing external data gracefully. No unhandled promise rejections or uncaught exceptions.

### Scenario 5: Service crash and recovery
```bash
docker compose stop backend
sleep 5
docker compose start backend
# Wait for health check
# Run traffic again
```
**Pass criteria**: Backend recovers. Frontend reconnects. No stale state.

### Scenario 6: Bandwidth restriction (slow network)
```bash
curl -s -X POST http://toxiproxy:8474/proxies/backend-proxy/toxics \
  -d '{"name":"bandwidth","type":"bandwidth","attributes":{"rate":10}}'
```
**Pass criteria**: System remains functional under constrained bandwidth. Timeouts are handled.

For each scenario, capture:
- HTTP status codes from test requests
- Container logs (last 100 lines per service, structured JSON parsed)
- Any unhandled exceptions, panic messages, or process exits
- Recovery time after fault removal

---

## Phase 5: Produce verification report

Generate a structured JSON report consumable by the feedback loop:

```json
{
  "dst_result": "PASS" | "FAIL" | "PARTIAL",
  "timestamp": "2026-03-30T12:00:00Z",
  "topology": { "...from Phase 1..." },
  "boot": {
    "status": "PASS",
    "startup_time_ms": 8500,
    "smoke_test": "PASS"
  },
  "scenarios": [
    {
      "name": "network_latency_fe_be",
      "status": "PASS" | "FAIL",
      "fault_type": "latency",
      "fault_target": "backend-proxy",
      "fault_params": {"latency": 2000},
      "test_results": {
        "http_status": 200,
        "response_time_ms": 2340,
        "unhandled_exceptions": 0,
        "error_logs": []
      },
      "recovery": {
        "status": "PASS",
        "recovery_time_ms": 150
      }
    }
  ],
  "summary": {
    "total_scenarios": 6,
    "passed": 5,
    "failed": 1,
    "failed_scenarios": ["db_timeout"],
    "critical_findings": [
      {
        "scenario": "db_timeout",
        "finding": "Connection pool exhausted after 5s DB timeout",
        "fix_category": "resilience",
        "suggested_action": "Add connection pool timeout config and circuit breaker to DB client",
        "evidence": {
          "log_line": "sqlalchemy.exc.TimeoutError: QueuePool limit reached",
          "container": "backend",
          "timestamp": "2026-03-30T12:01:23Z"
        }
      }
    ]
  }
}
```

---

## Cleanup

Always run cleanup, even on failure:
```bash
docker compose down -v --remove-orphans --timeout 30
docker network prune -f
```

---

## Structured output contract

The calling agent (verification-gate or generating agent) expects:
1. **On success**: Full JSON report at `./dst-report.json`
2. **On failure**: Partial report with `phase`, `status`, `error`, `fix_category`,
   `suggested_action` — enough for the agent to know what to fix
3. **Always**: `fix_category` is one of: `startup_error`, `integration_error`,
   `resilience`, `timeout_handling`, `error_handling`, `state_management`, `recovery`

## Reference files

- `references/toxiproxy-recipes.md` — Extended fault scenarios and toxic combinations
- `references/wiremock-stubs.md` — Common 3rd-party API mock patterns

## Dependent skills (not bundled, must be installed separately)

- `docker-patterns` — Install from `affaan-m/everything-claude-code`. Handles
  Dockerfile generation, compose structure, volumes, security, networking.
- `structured-logs` — Custom skill. Handles JSON logging setup, error categorization,
  trace ID propagation. Required for Phase 5 log parsing to work.
