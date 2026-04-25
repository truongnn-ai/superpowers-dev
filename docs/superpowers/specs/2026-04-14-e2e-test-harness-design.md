# E2E Test Harness Design

**Date:** 2026-04-14
**Status:** Approved
**Scope:** Enhance `subagent-driven-development` skill with a runtime test harness layer

---

## Problem

The current `subagent-driven-development` flow verifies code through static analysis only:
- Spec reviewer reads code and compares against requirements
- Code quality reviewer reads code and checks structure and style

Neither reviewer executes the code. Bugs that only manifest at runtime (integration failures, missing dependencies, wrong service wiring, broken E2E flows) survive all review stages and reach the user.

---

## Goal

Add a runtime test harness layer to the SDD flow that:
1. Executes tests in the actual runtime environment — not just reads them
2. Covers unit, integration, and E2E tiers based on task complexity
3. Works autonomously (no human intervention needed for normal pass/fail cycles)
4. Grounds test expectations in a pre-agreed contract, not ad-hoc reviewer judgment

---

## Design Overview

Two additions to the SDD flow:

1. **Implementation-Testing Contract (ITC)** — a structured agreement negotiated by two agents (coding agent + testing agent) before implementation begins, defining exactly what will be built and how it will be tested
2. **Test-runner subagent** — a pure executor that runs commands from the ITC and reports pass/fail/blocked status to the controller

---

## Section A: Two-Agent Contract Negotiation

### Participants

- **Coding agent** (`general-purpose`): reads task spec + existing codebase → proposes what will be built and natural test seams
- **Testing agent** (`general-purpose`): reads task spec + coding agent's proposal → challenges missing coverage, specifies exact test commands and acceptance criteria

Neither agent can finalize the contract alone. Both must sign off.

### Protocol

```
Round 1: coding-agent produces draft ITC
  - Components and files to be created
  - Known integration surfaces and external dependencies
  - Proposed test tier (unit / unit+integration)
  - "Cannot reasonably test X at runtime" declarations with reasoning

Round 2: testing-agent reviews draft
  - Accepts OR adds missing test cases
  - Specifies exact commands (not vague descriptions)
  - Challenges any "untestable" declarations if incorrect
  - Proposes acceptance criteria (counts, thresholds, required_services)

Round 3 (if needed): coding-agent responds to amendments
  - Accepts additions OR disputes with technical reasoning
  - Final response includes explicit ✅ or continued dispute

Termination:
  - testing-agent signs ✅ on coding-agent's final response → contract locked
  - 3 rounds without agreement → controller surfaces conflict to user
```

### Two Contract Scopes

| Contract | Created when | Covers |
|---|---|---|
| **Task ITC** | Before each task's implement loop | Unit + integration tests for that task |
| **Solution ITC** | After all tasks complete, before final review | E2E scenarios + full suite across the whole feature |

The solution ITC is created **after** all tasks complete because it must reference the actual implementation — real routes, components, and data flows that only exist once code is written. Creating it from the plan would produce speculative E2E contracts that may not match reality.

---

## Section B: Contract Structure

### Task ITC (`YYYY-MM-DDTHH-MM-SS-task_itc_N.md`)

```yaml
task_id: 2
task_name: "Auth middleware"

implementation_scope:
  components:
    - JWT validation middleware (src/middleware/auth.ts)
    - Error response formatter  (src/middleware/errors.ts)
    - Route protection wrapper  (src/routes/protected.ts)
  integration_surfaces:
    - Express route registration
    - DB token lookup (depends on Task 1 User model)

test_contract:
  tiers_required: [unit, integration]
  rationale: "Middleware touches DB for token lookup — unit tests cover
              logic branches, integration tests verify the full request
              cycle with a real database connection"

  unit:
    commands:
      - "npm test -- src/middleware/auth.test.ts"
    must_cover:
      - valid token accepted
      - expired token → 401
      - missing token → 401
      - malformed token → 400
      - DB failure handled gracefully

  integration:
    commands:
      - "npm test -- tests/integration/auth.test.ts"
    must_cover:
      - full request cycle with real DB
      - token validated against stored user
    required_services:
      - database (test instance)

acceptance_criteria:
  unit: all pass
  integration: all pass
  no_regression: true

sign_off:
  coding_agent: ✅
  testing_agent: ✅
```

**Unit-only example** (health check endpoint — no external dependencies):

```yaml
task_id: 1
task_name: "Health check endpoint"

test_contract:
  tiers_required: [unit]
  rationale: "Handler returns a static response with no external
              dependencies — no services to spin up, no DB calls"

  unit:
    commands:
      - "npm test -- src/routes/health.test.ts"
    must_cover:
      - returns 200 with { status: ok }
      - responds within 100ms

acceptance_criteria:
  unit: all pass
  no_regression: true

sign_off:
  coding_agent: ✅
  testing_agent: ✅
```

**Tier vocabulary — task ITC vs solution ITC are separate:**

Task ITC valid tiers: `unit`, `integration`
Solution ITC valid tiers: `e2e`, `full_suite`

These vocabularies do not overlap. A task ITC never declares `e2e`. A solution ITC never declares `unit` or `integration` — those were already verified per-task.

**`unit` is always the minimum tier for task ITCs.** Every task has unit tests. `integration` is added on top when the task crosses a service boundary.

**Tier selection guide:**

| Task type | Example | tiers_required |
|---|---|---|
| Isolated logic, no external deps | Health check, pure utility, data transformer | `[unit]` |
| Component with external boundary | API endpoint + DB, auth middleware | `[unit, integration]` |
| Frontend integrating with backend | Login form calling auth API | `[unit, integration]` |
| Full user journey (solution-level only) | Login → session → protected route | Solution ITC `[e2e, full_suite]` |

### Solution ITC (`YYYY-MM-DDTHH-MM-SS-solution_itc.md`)

```yaml
solution: "Auth System"
tasks_covered: [1, 2, 3, 4, 5]

implementation_summary:
  entry_points:
    - POST /api/auth/register
    - POST /api/auth/login
    - GET  /api/auth/refresh
    - POST /api/auth/logout
  protected_routes:
    - GET /api/users/profile
    - PUT /api/users/settings

test_contract:
  tiers_required: [e2e, full_suite]
  rationale: "All tasks complete — E2E scenarios verify the full user
              journey end-to-end; full suite confirms no cross-task
              regression was introduced"

  e2e:
    commands:
      - "npx playwright test tests/e2e/auth.spec.ts"
      - "npx playwright test tests/e2e/protected-routes.spec.ts"
    scenarios:
      - user registration + email verification
      - login and session persistence
      - protected route access with valid token
      - token expiry + refresh cycle
      - logout + session cleanup
      - unauthorized access redirects correctly
    required_services:
      - backend API (running)
      - database (seeded)
      - frontend (built)

  full_suite:
    command: "npm test"
    rationale: "Catch any cross-task regression not visible in individual task ITCs"

acceptance_criteria:
  e2e: all scenarios pass
  full_suite: all pass
  no_regression: true

sign_off:
  coding_agent: ✅
  testing_agent: ✅
```

### Contract File Location

```
docs/superpowers/contracts/
  2026-04-14T14-30-00-task_itc_1.md
  2026-04-14T14-32-15-task_itc_2.md
  2026-04-14T14-35-40-task_itc_3.md
  ...
  2026-04-14T16-20-00-solution_itc.md
```

Format: `YYYY-MM-DDTHH-MM-SS-<type>_<id>.md` — ISO 8601 with colons replaced by hyphens (filesystem-safe). Lexicographic sort = chronological order. Contracts are committed to git after each negotiation. If a contract is re-negotiated, the new file gets a new timestamp — the old one remains in git history as an audit trail.

---

## Section C: Enhanced SDD Flow

```
Read plan → extract all tasks → create TodoWrite
  ↓
For each task_i:

  ┌── Task ITC Negotiation ─────────────────────────────────────┐
  │  Round 1: coding-agent(task spec + codebase) → draft ITC    │
  │  Round 2: testing-agent(task spec + draft) → amendments     │
  │  Round 3 (if needed): coding-agent responds                 │
  │  → YYYY-MM-DDTHH-MM-SS-task_itc_N.md committed             │
  │  → Both sign ✅ or escalate to user                         │
  └─────────────────────────────────────────────────────────────┘
    ↓
  [Existing] Implement loop
    Task(implementer, spec + context + task_ITC path)
    break on DONE / DONE_WITH_CONCERNS
    ↓
  [Existing] Spec review loop
    Task(spec-reviewer, requirements + task_ITC + implementer_report)
    break on ✅, else implementer fixes
    ↓
  [Existing] Code review loop
    Task(code-reviewer, BASE_SHA + HEAD_SHA + task_ITC)
    break on ✅, else implementer fixes
    ↓
  [NEW] Unit test harness
    while true:
      Task(test-runner, unit commands from task_ITC)
      break on PASS
      FAIL → implementer fixes → re-run
      BLOCKED → resolve environment, re-run
    ↓
  [NEW] if tiers_required includes [integration]:
    while true:
      Task(test-runner, integration commands from task_ITC)
      break on PASS
      FAIL → implementer fixes → re-run
      BLOCKED → resolve environment, re-run
  ↓
  Mark task complete → next task

↓
[After all tasks]

  ┌── Solution ITC Negotiation ─────────────────────────────────┐
  │  coding-agent scans ACTUAL implementation (routes, etc.)    │
  │  Round 1: coding-agent → draft solution ITC                 │
  │  Round 2: testing-agent → E2E scenarios + full suite cmds   │
  │  Round 3 (if needed): coding-agent responds                 │
  │  → YYYY-MM-DDTHH-MM-SS-solution_itc.md committed           │
  │  → Both sign ✅ or escalate to user                         │
  └─────────────────────────────────────────────────────────────┘
    ↓
  [NEW] E2E + full suite harness
    while true:
      Task(test-runner-solution, solution_ITC e2e + full_suite cmds)
      break on PASS
      FAIL → implementer fixes → re-run harness
      BLOCKED → resolve environment (stack not running, etc.)
    ↓
  [Existing] Final code review
    Task(superpowers:code-reviewer, full implementation)
    ↓
  finishing-a-development-branch
```

**Why unit harness runs after code review, not before:**
Code reviewer catches structural issues (missing error handling, wrong return types) that would cause test failures. Fixing code-level issues first is cheaper than spinning up services, running integration tests, and debugging runtime failures caused by structural problems.

**Why unit and integration run as separate sequential loops:**
If unit tests fail, the component itself is broken — there is no point running integration tests. Fix unit first, then verify the boundary. Isolated failures, cheaper feedback loop.

**Why E2E failures re-run the harness, not renegotiate the solution ITC:**
The solution ITC was written against the actual implementation. If E2E fails, the code is wrong — not the contract. Renegotiating the contract to match broken code lowers the bar rather than fixing the problem.

---

## Section D: Test-Runner Subagent

### Core Principle

Pure executor. Runs commands, captures output, reports structured results. Does not fix, does not judge, does not interpret. The controller decides what to do with failures.

### Two Prompt Templates

**`test-runner-task-prompt.md`** — for per-task tiers (unit / integration):

```
Task tool (general-purpose):
  description: "Run [unit|integration] tests for Task N"
  prompt: |
    You are a test runner. Execute these commands exactly and report results.
    Do NOT fix anything. Do NOT modify any files.

    ## Commands to Run
    [commands from task_ITC for this tier]

    ## Required Services
    [task_ITC.test_contract.<tier>.required_services OR "none"]

    ## Acceptance Criteria
    [task_ITC.acceptance_criteria for this tier]

    ## Your Job
    1. Check required services are available (if any declared)
    2. Run each command in order, capturing full stdout + stderr
    3. Report results — do NOT fix, do NOT modify files

    ## Report Format
    Status: PASS | FAIL | BLOCKED

    Results:
      - command: "<command>"
        status: PASS | FAIL
        summary: "<X/Y tests passed>"
        failures: (if FAIL)
          - test: "<test name>"
            error: "<error message>"
            output: [relevant log lines]

    BLOCKED if:
    - Required service not running
    - Missing env vars
    - Tool not installed (playwright, jest, etc.)
    - Test environment not set up
    Report BLOCKED with reason — do NOT attempt to fix.
```

**`test-runner-solution-prompt.md`** — for solution-level (E2E + full suite):

```
Task tool (general-purpose):
  description: "Run E2E and full suite for solution"
  prompt: |
    You are a test runner. Execute these commands exactly and report results.
    Do NOT fix anything. Do NOT modify any files.

    ## Required Stack
    Verify before running:
    [solution_ITC.test_contract.e2e.required_services]
    If any service is not running, report BLOCKED immediately.

    ## Commands to Run
    E2E: [solution_ITC.test_contract.e2e.commands]
    Full suite: [solution_ITC.test_contract.full_suite.command]

    ## E2E Scenarios to Verify
    [solution_ITC.test_contract.e2e.scenarios — list each]

    ## Your Job
    1. Verify full stack is running (API + DB + frontend built)
    2. Run E2E commands, capture which scenarios passed/failed
    3. Run full suite command, capture pass/fail counts
    4. Report results — do NOT fix, do NOT modify files

    ## Report Format
    Status: PASS | FAIL | BLOCKED

    E2E Results:
      - scenario: "<scenario name>"
        status: PASS | FAIL
        failure_detail: (if FAIL) what user action failed and why

    Full Suite:
      status: PASS | FAIL
      summary: "<X/Y tests passed>"
      failures: (if FAIL) list with file:line

    BLOCKED if: required service not running, tool not installed, build missing.
    Report BLOCKED with reason — do NOT attempt to fix.
```

### Output Status → Controller Action

| Status | Meaning | Controller action |
|---|---|---|
| `PASS` | All commands exited 0, criteria met | Exit harness loop, proceed |
| `FAIL` | Commands ran, tests failed | Extract failures → implementer fixes → re-run harness |
| `BLOCKED` | Commands could not run | Resolve environment issue → re-run (do not dispatch implementer) |

`BLOCKED` is strictly distinct from `FAIL`. BLOCKED = environment problem (controller resolves). FAIL = code problem (implementer resolves).

---

## What This Does NOT Change

- Spec reviewer and code quality reviewer behavior is unchanged — they continue static analysis
- The implementer's self-review step is unchanged
- The test-runner does not replace the reviewers; it adds an independent execution layer after them
- Unit test design is still verified by the code reviewer (tests exist, cover right cases) — the harness adds runtime execution verification on top
