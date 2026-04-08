---
name: verification-gate
description: Orchestrate multi-signal verification of generated code and produce structured feedback for the generating agent. Use when: code generation is complete and needs autonomous verification, running the full verification pipeline, checking if generated code is ready for deployment, or when the user says "verify everything","run verification", "is this code ready", "check the generated code", "run the gate", "quality check", or "feedback loop". This skill coordinates static analysis, property-based tests, mutation testing, contract validation, and DST simulation into a single pass/fail decision with actionable fix instructions. Use this skill AFTER code generation and BEFORE any deployment or merge.
context: fork
---

# Verification Gate

The orchestrator skill for the autonomous feedback loop. Runs all verification
signals in dependency order, aggregates results, and produces a single structured
report that tells the generating agent exactly what to fix.

## Pipeline overview

Signals run in order from cheapest to most expensive. If a signal with
`stop_on_fail: true` fails, skip remaining signals — fix the cheap stuff first.

```
Signal 1: Static Analysis     [~5s]   stop_on_fail: true
Signal 2: Property Tests       [~30s]  stop_on_fail: true
Signal 3: Mutation Testing     [~2min] stop_on_fail: false
Signal 4: Contract Validation  [~10s]  stop_on_fail: true
Signal 5: DST Simulation       [~3min] stop_on_fail: false
Signal 6: E2E Tests            [~1-3min] stop_on_fail: true
```

Why this order: Static analysis catches syntax/type/security issues in seconds.
Property tests catch logic errors in 30 seconds. No point running a 3-minute
Docker simulation if the code doesn't even type-check.

## Per-Task Mode

When invoked per-task (by `docker-verified-execution`), verification-gate accepts a **profile** — a subset of signals to run. This avoids running expensive signals on tasks that don't need them.

**Invocation:**
- Full pipeline (default): run all 6 signals in order
- Per-task mode: receive `signals: [containers, lint, e2e]` from the task's verification profile
- Always run `containers` first (health check gate) regardless of profile
- Skip signals not in the profile
- Cost ordering preserved within the subset

**Profile source:** Each task in the implementation plan is tagged with a verification profile (see `verification-profiles` skill). The profile expands to a signal list.

**Convergence tracking:** Scoped per-task in per-task mode. Error signatures reset between tasks.

**Assume Docker running:** In per-task mode, Docker containers are already running (set up in Task 0 by `docker-patterns`). Do NOT boot Docker — just verify containers are healthy via `docker compose ps`.

---

## Signal 1: Static Analysis

Run language-appropriate linters, type checkers, and security scanners.

### Python
```bash
ruff check . --output-format json > /tmp/ruff-out.json 2>&1 || true
mypy --strict . --no-error-summary 2>&1 | head -50 > /tmp/mypy-out.txt || true
bandit -r src/ -f json -o /tmp/bandit-out.json 2>&1 || true
```

### TypeScript / JavaScript
```bash
npx tsc --noEmit 2>&1 | head -50 > /tmp/tsc-out.txt || true
npx eslint . --format json -o /tmp/eslint-out.json 2>&1 || true
npm audit --json > /tmp/npm-audit.json 2>&1 || true
```

### Go
```bash
go vet ./... 2>&1 | head -50 > /tmp/govet-out.txt || true
golangci-lint run --out-format json > /tmp/golint-out.json 2>&1 || true
```

Parse each tool's output. Classify findings:
- `critical`: type errors, security vulnerabilities (bandit HIGH/MEDIUM), unresolved imports
- `warning`: style issues, unused variables, deprecated APIs
- `info`: formatting, naming conventions

**Stop condition**: Any `critical` finding → stop pipeline, return feedback.

---

## Signal 2: Property-Based Tests

Look for property test files:
- Python: `test_properties_*.py`, `**/test_*_properties.py`
- JS/TS: `*.property.test.ts`, `*.property.test.js`
- Go: `*_property_test.go`

If found, run them:
```bash
# Python
pytest test_properties_*.py -x --tb=short -q --hypothesis-seed=0 2>&1 | tee /tmp/pbt-out.txt

# JS/TS
npx vitest run --reporter=json "**/*.property.test.*" 2>&1 | tee /tmp/pbt-out.json

# Go
go test -run Property ./... -count=1 -v 2>&1 | tee /tmp/pbt-out.txt
```

If NO property tests exist, emit a warning — do not skip silently:
```json
{
  "signal": "property_tests",
  "status": "WARN",
  "message": "No property tests found. Unit tests alone cannot be trusted in an autonomous pipeline. Generate property tests from the spec before proceeding.",
  "fix_category": "missing_specification",
  "suggested_action": "Use trailofbits/property-based-testing skill to generate properties from the PRD/spec"
}
```

**Stop condition**: Any property test failure → stop pipeline. Property failures
are specification violations — the code doesn't do what it should.

---

## Signal 3: Mutation Testing

Validates that the tests actually catch bugs (not just pass on correct code).

See `references/mutation-testing.md` for full setup per language.

### Quick reference
```bash
# Python
mutmut run --paths-to-mutate=src/ --tests-dir=tests/ --runner="pytest -x -q" --no-progress 2>&1
mutmut results > /tmp/mutmut-out.txt

# JS/TS
npx stryker run --reporters json 2>&1
cat reports/mutation/mutation.json > /tmp/stryker-out.json

# Go
go-mutesting ./... 2>&1 | tee /tmp/gomut-out.txt
```

Parse mutation score: `killed / total`.

Thresholds:
- Code touching money, auth, data integrity → score < 60% is `FAIL`
- Standard business logic → score < 40% is `FAIL`
- Experimental / non-critical → score < 20% is `FAIL`

For surviving mutants, extract the top 5 most impactful:
- File, line, mutation type, original vs mutated code
- `fix_category: "test_gap"`
- `suggested_action: "Add test covering boundary: <description of mutation>"`

**Stop condition**: Never stops the pipeline. Mutation results are advisory —
they tell the agent to strengthen tests, not that the code is wrong.

---

## Signal 4: Contract Validation

Check that FE and BE agree on API shapes.

### If OpenAPI spec exists
```bash
# Validate BE responses against spec
npx @apidevtools/swagger-cli validate openapi.yaml 2>&1 | tee /tmp/openapi-validate.txt

# If using Schemathesis (Python)
schemathesis run openapi.yaml --base-url http://localhost:8000 --checks all \
  --hypothesis-max-examples=50 2>&1 | tee /tmp/schemathesis-out.txt
```

### If Pact contracts exist
```bash
# Run consumer-driven contract tests
npx pact-provider-verifier --provider-base-url=http://localhost:8000 \
  --pact-urls=./pacts/*.json 2>&1 | tee /tmp/pact-out.txt
```

### If neither exists
Check for type mismatches manually:
1. Find all FE fetch/axios calls → extract expected response shapes
2. Find all BE route handlers → extract actual response shapes
3. Compare field names and types
4. Report mismatches as `fix_category: "contract_mismatch"`

**Stop condition**: Any contract mismatch → stop pipeline. FE/BE disagreement
means the app is broken even if individual tests pass.

---

## Signal 5: DST Simulation

Invoke the `dst-simulation` skill. This is the most expensive signal — only
runs if signals 1-4 pass (or pass with warnings).

```
/dst-simulation
```

The DST skill produces `dst-report.json`. Read it and merge into the
unified verification report.

If DST is not available (no Docker, no compose file), skip with:
```json
{
  "signal": "dst_simulation",
  "status": "SKIP",
  "reason": "Docker not available or no compose configuration found"
}
```

---

## Signal 6: E2E Tests (~1-3 min)

Run Playwright end-to-end tests against Docker containers.

**When to run:** Signal `e2e` is in the task's profile.

**Prerequisites:** All containers healthy (Signal `containers` passed).

**Execution:**
```bash
# Per-task: run specific Playwright project based on profile
npx playwright test --project=Figma-Compliance    # for ui-component profile
npx playwright test --project=Business-Workflows  # for integration profile
npx playwright test --project=Smoke               # for quick verification

# Final verification: run all projects
npx playwright test
```

**Configuration:**
- `baseURL`: `http://localhost:<PORT>` (Docker-exposed port)
- Animations disabled for visual regression stability
- Screenshots on failure saved to `test-results/`

**Output format:**
```json
{
  "signal": "e2e",
  "status": "fail",
  "duration_seconds": 45,
  "project": "Figma-Compliance",
  "summary": { "passed": 8, "failed": 2, "skipped": 0 },
  "failures": [
    {
      "test": "homepage.figma.spec.ts > nav background color matches Figma token",
      "expected": "rgb(28, 27, 27)",
      "actual": "rgb(255, 255, 255)",
      "screenshot_diff": "test-results/nav-bg-diff.png"
    }
  ]
}
```

**stop_on_fail:** true — e2e failures indicate visible user-facing bugs.

**Fix routing:** E2e failures route to the generating agent with the failure details and screenshot diffs. The agent reads structured logs (`docker compose logs`) for backend errors that may cause frontend failures.

---

## Unified output format

Produce ONE file: `verification-report.json`

```json
{
  "verification_result": "PASS | FAIL | PARTIAL",
  "timestamp": "2026-03-30T12:00:00Z",
  "duration_ms": 245000,
  "retry_number": 1,
  "max_retries": 5,

  "signals": [
    {
      "name": "static_analysis",
      "status": "PASS | FAIL | WARN | SKIP",
      "duration_ms": 4200,
      "critical_count": 0,
      "warning_count": 3,
      "findings": []
    },
    {
      "name": "property_tests",
      "status": "PASS",
      "duration_ms": 28000,
      "total": 15,
      "passed": 15,
      "failed": 0,
      "failures": []
    },
    {
      "name": "mutation_testing",
      "status": "WARN",
      "duration_ms": 120000,
      "mutation_score": 0.58,
      "killed": 45,
      "survived": 32,
      "top_survivors": []
    },
    {
      "name": "contract_validation",
      "status": "PASS",
      "duration_ms": 8000,
      "mismatches": []
    },
    {
      "name": "dst_simulation",
      "status": "FAIL",
      "duration_ms": 180000,
      "scenarios_passed": 4,
      "scenarios_failed": 2,
      "failures": []
    }
  ],

  "feedback": {
    "total_issues": 3,
    "by_category": {
      "resilience": 2,
      "test_gap": 1
    },
    "actions": [
      {
        "priority": 1,
        "fix_category": "resilience",
        "signal_source": "dst_simulation",
        "description": "Backend crashes on DB timeout — no circuit breaker",
        "file": "src/db/client.py",
        "line": 42,
        "suggested_action": "Add connection pool timeout (5s) and circuit breaker pattern to DB client",
        "evidence": "DST scenario 'db_timeout' — backend exited with code 1"
      },
      {
        "priority": 2,
        "fix_category": "resilience",
        "signal_source": "dst_simulation",
        "description": "Frontend shows raw error object instead of user-friendly message on API failure",
        "file": "src/pages/dashboard.tsx",
        "line": 88,
        "suggested_action": "Add error boundary and fallback UI for API failure states",
        "evidence": "DST scenario 'reset_fe_be' — frontend rendered [object Object]"
      },
      {
        "priority": 3,
        "fix_category": "test_gap",
        "signal_source": "mutation_testing",
        "description": "Boundary condition in pricing calculation not tested",
        "file": "src/pricing.py",
        "line": 67,
        "suggested_action": "Add property test: quantity == 0 should return 0, not negative",
        "evidence": "Mutant survived: changed '>' to '>=' on line 67"
      }
    ],
    "primary_instruction": "Fix the 2 resilience issues first (DB circuit breaker + FE error boundary), then add the missing property test. Re-run /verification-gate after fixes."
  },

  "convergence": {
    "is_stuck": false,
    "repeated_errors": [],
    "recommendation": "retry"
  }
}
```

---

## Retry and convergence logic

### Retry rules
After producing the report, evaluate whether to recommend retry:

1. **All signals PASS** → `verification_result: "PASS"`, no retry needed
2. **Only WARN signals** (mutation score low, but no failures) → `"PARTIAL"`, recommend retry to strengthen tests
3. **Any FAIL signal** → `"FAIL"`, recommend retry with fix actions
4. **retry_number >= max_retries** → stop, output report for human review

### Convergence detection
Track error signatures across retries. An error signature is:
`{signal}:{file}:{line}:{fix_category}`

If the same signature appears in 3 consecutive retries:
```json
{
  "convergence": {
    "is_stuck": true,
    "repeated_errors": [
      {
        "signature": "dst_simulation:src/db/client.py:42:resilience",
        "occurrences": 3,
        "recommendation": "escalate_to_human"
      }
    ],
    "recommendation": "stop — agent cannot resolve this issue autonomously"
  }
}
```

### Priority ordering
Actions are sorted by:
1. `static_analysis` failures first (cheapest to fix, blocks everything)
2. `property_tests` failures (specification violations)
3. `contract_validation` failures (integration boundaries)
4. `dst_simulation` failures (resilience issues)
5. `mutation_testing` warnings (test quality improvements)

---

## How this skill connects to others

```
structured-logs skill
  → Ensures generated code has parseable logs
  → DST reads these logs to produce its report

dst-simulation skill
  → Runs fault injection scenarios
  → Produces dst-report.json consumed by this skill

trailofbits/property-based-testing (community)
  → Guides PBT generation
  → This skill runs and evaluates the resulting tests

obra/verification-before-completion (community)
  → Enforces "prove it works before claiming done"
  → This skill IS the proof mechanism
```

---

## Invoking this skill

The generating agent calls:
```
/verification-gate
```

Or programmatically in a LangGraph pipeline:
1. Read `verification-report.json`
2. If `verification_result == "PASS"` → proceed to deploy
3. If `verification_result == "FAIL"` → feed `feedback.actions` back to generating agent
4. If `convergence.is_stuck == true` → exit loop, flag for human

---

## Reference files

- `references/mutation-testing.md` — Setup and execution per language
- `references/signal-routing.md` — Decision tree for fix strategy per failure type
