---
name: docker-verified-execution
description: "Use when executing implementation plan tasks that must be verified against running Docker containers — orchestrates the deploy-test-diagnose-fix loop (Ralph Loop) per task with structured log diagnosis and convergence detection"
---

# Docker-Verified Execution

The deploy → test → diagnose → fix loop for each task. Every code change is verified against running Docker containers before committing.

## When to Use

During plan execution (subagent-driven-development or executing-plans). Wraps each task with Docker verification.

**Prerequisites:**
- Docker environment running (Task 0 completed via docker-patterns)
- Task tagged with a verification profile (via verification-profiles skill)
- Structured logging configured (via structured-logs skill)

## The Ralph Loop

Never let the agent judge its own fix. Docker and the test runner are the arbiters.

```
┌──────────────────────────┐
│  1. WRITE CODE           │
│     (TDD skill)          │
│     - Write failing test │
│     - Implement          │
│     - Local tests pass   │
└───────────┬──────────────┘
            ▼
┌──────────────────────────┐
│  2. DEPLOY TO DOCKER     │
│                          │
│  Option A (fast):        │
│    docker compose watch  │
│    → file sync, no build │
│                          │
│  Option B (rebuild):     │
│    docker compose up     │
│      --build <svc> -d    │
│                          │
│  Wait healthy:           │
│    poll docker compose ps│
│    timeout: 120s         │
└───────────┬──────────────┘
            ▼
┌──────────────────────────┐
│  3. RUN VERIFICATION     │
│     verification-gate    │
│     with task's profile  │
│                          │
│  e.g., signals:          │
│  [containers, lint, e2e] │
└───────────┬──────────────┘
            ▼
┌──────────────────────────┐
│  4. ALL PASS?            │
│  ├─ YES → commit,       │
│  │        next task      │
│  └─ NO  → step 5        │
└───────────┬──────────────┘
            ▼
┌──────────────────────────┐
│  5. DIAGNOSE             │
│                          │
│  Read structured logs:   │
│  docker compose logs     │
│    --no-log-prefix       │
│    --tail 100 <service>  │
│    | jq 'fromjson?       │
│      // empty            │
│      | select(.level ==  │
│        "error")'         │
│                          │
│  Categorize errors:      │
│    error_category field  │
│    → fix strategy        │
│                          │
│  Invoke systematic-      │
│  debugging with evidence │
└───────────┬──────────────┘
            ▼
┌──────────────────────────┐
│  6. LOOP GUARD           │
│                          │
│  Hash current errors:    │
│    sort error messages   │
│    → md5 hash            │
│                          │
│  Same hash as last try?  │
│  → STUCK, escalate       │
│                          │
│  Attempt >= 5?           │
│  → EXHAUSTED, escalate   │
│                          │
│  Different errors?       │
│  → PROGRESS, loop to 1  │
└──────────────────────────┘
```

## Exit Conditions

| Condition | Action |
|---|---|
| All verification signals pass | Commit changes, proceed to next task |
| Same errors 2 consecutive attempts | STUCK — escalate to user with error details and log excerpts |
| 5 attempts exhausted | EXHAUSTED — escalate to user with full history of attempts |
| Container won't start | INFRASTRUCTURE — escalate immediately, don't retry code changes |

## Log Querying Patterns

```bash
# All errors from a service
docker compose logs --no-log-prefix --tail 100 backend \
  | jq -R 'fromjson? // empty | select(.level == "error")'

# Filter by trace_id (follow a single request)
docker compose logs --no-log-prefix backend \
  | jq -R 'fromjson? // empty | select(.trace_id == "abc123")'

# Count errors by category (find dominant failure)
docker compose logs --no-log-prefix backend \
  | jq -R 'fromjson? // empty | select(.level == "error") | .error_category' \
  | sort | uniq -c | sort -rn

# Errors from last 5 minutes only
docker compose logs --no-log-prefix --since 5m backend \
  | jq -R 'fromjson? // empty | select(.level == "error")'

# Cross-service trace (follow request across services)
TRACE_ID="abc123"
docker compose logs --no-log-prefix \
  | jq -R 'fromjson? // empty | select(.trace_id == "'$TRACE_ID'")' \
  | jq -s 'sort_by(.timestamp)'
```

## Error Category → Fix Strategy

Uses the `error_category` field from structured-logs:

| Category | Meaning | Fix Strategy |
|---|---|---|
| `timeout` | Request exceeded time limit | Check query performance, add indices, increase timeout |
| `connection` | Failed upstream connection | Verify service networking, check Docker compose links |
| `validation` | Input/output schema mismatch | Fix request/response shapes, update types |
| `logic` | Business rule violation | Fix implementation logic |
| `auth` | Auth/authz failure | Fix token handling, check credentials config |
| `rate_limit` | Rate limit exceeded | Add backoff, check test parallelism |
| `unknown` | Unhandled exception | Read stack trace, add error handling |

## Deploy Strategy Selection

| Scenario | Strategy | Command |
|---|---|---|
| Source file changed (hot reload) | Watch/sync | `docker compose watch` (already running) |
| Dependency changed (package.json, requirements.txt) | Rebuild service | `docker compose up --build <service> -d` |
| Docker config changed (Dockerfile, compose) | Full rebuild | `docker compose up --build -d` |
| Database schema changed | Rebuild + migrate | `docker compose up --build -d && docker compose exec backend alembic upgrade head` |

## Health Check Polling

```bash
wait_healthy() {
  local timeout=120 elapsed=0 interval=5
  while [ $elapsed -lt $timeout ]; do
    unhealthy=$(docker compose ps --format json \
      | jq -r '.[] | select(.Health != "healthy") | .Service' 2>/dev/null)
    if [ -z "$unhealthy" ]; then
      echo "All services healthy after ${elapsed}s"
      return 0
    fi
    echo "Waiting for: $unhealthy (${elapsed}s/${timeout}s)"
    sleep $interval
    elapsed=$((elapsed + interval))
  done
  echo "TIMEOUT: services still unhealthy after ${timeout}s"
  docker compose logs --tail 20
  return 1
}
```

## Integration with Other Skills

| Skill | Role in the Loop |
|---|---|
| `test-driven-development` | Step 1 — write code with TDD |
| `docker-patterns` | Step 2 — deploy strategy, health checks |
| `verification-gate` | Step 3 — run task's verification signals |
| `structured-logs` | Step 5 — parse container logs for diagnosis |
| `systematic-debugging` | Step 5 — root cause investigation from log evidence |
| `verification-profiles` | Provides signal list per task |
