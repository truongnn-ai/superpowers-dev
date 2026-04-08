# Signal Routing Decision Tree

When the verification gate produces a failure, the generating agent needs to know
HOW to fix it — not just WHAT broke. This reference maps failure signals to
concrete fix strategies.

---

## Routing by fix_category

### `startup_error`
**Source**: Signal 1 (static analysis) or Signal 5 (DST boot phase)
**Cause**: Code doesn't compile, missing dependencies, bad config
**Fix strategy**:
1. Read the exact error message
2. If import error → add dependency or fix import path
3. If type error → fix the type mismatch (mypy/tsc output has the exact location)
4. If config error → check environment variables and connection strings
**Retry expectation**: Usually fixed in 1 retry

### `logic`
**Source**: Signal 2 (property tests)
**Cause**: Business rule violation — code produces wrong output
**Fix strategy**:
1. Read the minimal failing input from Hypothesis/fast-check shrinking
2. Trace the code path for that specific input
3. Fix the conditional/algorithm that produces wrong output
4. Do NOT change the property test — it represents the specification
**Retry expectation**: 1-2 retries

### `validation`
**Source**: Signal 2 (property tests) or Signal 4 (contract validation)
**Cause**: Input/output shape doesn't match expected schema
**Fix strategy**:
1. Compare expected schema (from contract/spec) with actual output
2. Add missing fields, fix types, handle null/undefined cases
3. Add input validation at API boundaries
**Retry expectation**: 1 retry

### `contract_mismatch`
**Source**: Signal 4 (contract validation)
**Cause**: FE expects different data shape than BE provides
**Fix strategy**:
1. Determine which side is "source of truth" (usually the spec/OpenAPI)
2. If BE is wrong → fix the serializer/response builder
3. If FE is wrong → fix the API client types/interfaces
4. If neither matches spec → fix both
**Retry expectation**: 1 retry

### `test_gap`
**Source**: Signal 3 (mutation testing)
**Cause**: Tests exist but don't actually verify the behavior
**Fix strategy**:
1. Read the surviving mutant description
2. Write a NEW test (preferably property-based) that would catch this mutation
3. Do NOT modify existing code — the code might be correct, tests are weak
4. Re-run mutation testing to confirm the mutant is killed
**Retry expectation**: 1 retry per mutant batch

### `timeout`
**Source**: Signal 5 (DST simulation)
**Cause**: Service doesn't handle upstream timeouts gracefully
**Fix strategy**:
1. Add explicit timeout configuration to the HTTP/DB client
2. Add circuit breaker pattern (fail fast after N consecutive failures)
3. Add retry with exponential backoff for transient failures
4. Return appropriate HTTP status (504 for upstream timeout, 503 for circuit open)
5. Log with `error_category: "timeout"` (structured-logs skill)
**Retry expectation**: 1-2 retries

### `connection`
**Source**: Signal 5 (DST simulation)
**Cause**: Service crashes or shows unhandled errors when upstream is unreachable
**Fix strategy**:
1. Wrap all outgoing calls in try/catch with specific error handling
2. Add health check endpoints that verify upstream connectivity
3. Add graceful degradation (return cached data, show fallback UI)
4. Log with `error_category: "connection"` (structured-logs skill)
**Retry expectation**: 1-2 retries

### `resilience`
**Source**: Signal 5 (DST simulation)
**Cause**: Service doesn't recover after a fault is removed
**Fix strategy**:
1. Check connection pool settings (are connections being returned?)
2. Add reconnection logic for persistent connections (WebSocket, DB pool)
3. Add liveness probe that restarts the service if it's unhealthy
4. Verify no in-memory state becomes corrupted during fault
**Retry expectation**: 2-3 retries (hardest category)

### `auth`
**Source**: Signal 1 (static analysis / bandit) or Signal 5 (DST)
**Cause**: Security issue in authentication/authorization
**Fix strategy**:
1. Never hardcode secrets — use environment variables
2. Validate tokens on every request (middleware, not per-route)
3. Use constant-time comparison for token validation
4. Log with `error_category: "auth"` but NEVER log the credential
**Retry expectation**: 1 retry

### `missing_specification`
**Source**: Signal 2 (property tests — when none exist)
**Cause**: No property tests → no specification → can't verify correctness
**Fix strategy**:
1. Read the PRD/spec/user story
2. Extract invariants (what must ALWAYS be true?)
3. Generate property-based tests using trailofbits/property-based-testing patterns
4. Re-run the verification gate
**Retry expectation**: 1 retry (but may need human input for complex specs)

### `unknown`
**Source**: Any signal — unhandled exception
**Cause**: Error the agent doesn't know how to categorize
**Fix strategy**:
1. Add proper error handling (specific catch blocks, not catch-all)
2. Categorize the error into one of the known categories
3. Log with appropriate `error_category`
4. If genuinely unknown, escalate to human
**Retry expectation**: 1-2 retries, then escalate

---

## Routing by signal source

When multiple signals fail, prioritize by signal order:

```
Signal 1 failures → fix first (blocks everything downstream)
Signal 2 failures → fix second (specification violations)
Signal 4 failures → fix third (integration boundaries)
Signal 5 failures → fix fourth (resilience)
Signal 3 warnings → fix last (test quality, non-blocking)
```

Within the same signal, prioritize by:
1. `critical` severity over `warning`
2. Fewer affected files first (quick wins)
3. `logic` and `security` categories over `style` and `test_gap`

---

## Escalation rules

Escalate to human review when:
- Same error signature repeats 3+ times across retries
- `unknown` category persists after 2 retries
- Mutation score stays below threshold after test improvements
- DST scenario requires architectural change (e.g., "add message queue")
- Contract mismatch requires spec clarification (ambiguous requirements)

Escalation format:
```json
{
  "escalation": true,
  "reason": "Agent stuck on resilience issue — requires architectural decision",
  "context": "DB circuit breaker pattern conflicts with existing transaction handling",
  "options": [
    "Add connection pool with timeout (simple, may miss edge cases)",
    "Add async retry queue (complex, handles all cases)",
    "Accept the risk and add monitoring alert instead"
  ],
  "blocking_signal": "dst_simulation",
  "retry_count": 3
}
```
