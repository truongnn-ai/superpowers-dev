# E2E Coverage Matrix — Design Spec

**Date:** 2026-04-20
**Branch:** feature/enhanced_harness
**Scope:** superpowers plugin — E2E test quality (Issues 1 and 2 from `discussion.md`)
**Out of scope:** context-length and token-cost issues (Issues 3 and 4) — to be addressed in a separate spec

## 1. Problem

The current superpowers harness has two linked weaknesses at the solution (end-to-end) tier:

1. **Missing scenarios.** Apps pass every task-level unit and integration test, yet still fail in end-to-end use. The failures cluster in two places:
   - **Feature interactions** — Feature A passes in isolation, Feature B passes in isolation, running them in sequence breaks state.
   - **Multi-step user journeys** — login → configure → use-feature flows fall apart mid-way (state loss, session bugs, navigation traps).

2. **Thin testing-agent playbook.** The testing-agent's current rubric is "happy path covered? error cases covered?". This produces shallow scenarios ("assert HTTP 200", "assert element visible") that don't catch the bugs above.

Both weaknesses share one root cause: **no one owns the cross-task user-journey view.** Each task is designed, implemented, and tested in isolation; the only place cross-task behavior is considered is the free-form `scenarios: [...]` list inside the solution-ITC, authored once at the very end of the run by agents that have no systematic prompt for enumerating combinations.

## 2. Goals and non-goals

**Goals**
- Make cross-task user journeys a first-class artifact of the spec, produced at brainstorm time.
- Replace free-form end-of-run scenario authoring with a mechanically-enforced **Journey × Strategy coverage matrix**.
- Give the testing-agent a codified playbook of test strategies that it must walk for every journey, with an explicit N/A escape hatch for genuine inapplicability.
- Keep all changes additive: no changes to task-level ITC, task iteration, code-review, or spec-review loops.

**Non-goals (this spec)**
- Incremental / mid-stream E2E execution — the solution-level runner continues to run once, at the end.
- Visual regression, accessibility, idempotency, concurrency, and boundary-value testing strategies — deferred to a follow-up spec that extends `testing-strategies.md`.
- Context length and token-cost reduction (Issues 3 and 4 in `discussion.md`) — separate spec.

## 3. Architecture overview

Two new artifacts, six file modifications, all additive:

**New artifacts**
- **`docs/superpowers/specs/<date>-<topic>-journeys.yaml`** — per-project structured journey list, peer to the design doc. Produced by brainstorming, consumed by writing-plans and by the solution-level coding-agent and testing-agent.
- **`skills/subagent-driven-development/testing-strategies.md`** — skill-level playbook. One-time artifact, shared across all projects, grows as new strategies are added.

**Workflow changes (see Section 7 for full file list)**
- `brainstorming` — design gate now includes Journey Enumeration as a required sub-step; spec-review covers `journeys.yaml` alongside the design doc.
- `writing-plans` — plan schema gains an optional `contributes_to: [J1, J3]` field per task and a top-level `## Journeys (reference)` section.
- `subagent-driven-development` — solution-ITC negotiation now produces a coverage matrix, not a scenarios list; the coding-agent, testing-agent, and solution-runner prompts are all updated to speak in matrix shape.

**Unchanged**
- Task-level ITC (format, negotiation rounds, escalation).
- Task iteration, code-review loop, spec-review loop, per-task unit/integration runner.
- Solution-ITC's adversarial 2-agent negotiation structure and 5-round escalation logic.

## 4. Journey artifact — `<topic>-journeys.yaml`

Structured, machine-readable, authored at brainstorm time, committed to the repo alongside the design doc.

```yaml
journeys:
  - id: J1
    name: "Signup → verify email → create workspace → invite member"
    persona: "new user, no prior account"
    preconditions: "empty DB, email provider configured"
    steps:
      - "POST /signup with valid email+password"
      - "Click verification link from email"
      - "POST /workspaces to create one"
      - "POST /workspaces/:id/invites with member email"
      - "Member opens invite link, accepts"
    expected_outcome: "Both users can access workspace, roles set correctly"
    feature_interactions: [auth, workspace, email, invites]
    priority: P0
```

**Field reference**
- `id` — stable identifier (e.g., `J1`, `J2`). Referenced from task plans (`contributes_to:`) and from the coverage matrix.
- `name` — human-readable label.
- `persona` — who is taking the journey, and any relevant attribute of that actor.
- `preconditions` — system state before the journey begins.
- `steps` — ordered actions the actor takes; specific enough that a tester can translate each into a runnable operation.
- `expected_outcome` — concrete end-state, including DB, UI, and side effects.
- `feature_interactions` — tags drawn from the design's feature list. Two journeys sharing a tag are candidates for a `feature_interaction` cell in the coverage matrix.
- `priority` — `P0` (must cover), `P1` (should cover), `P2` (nice to have). Used as a cost lever: P0 journeys get full strategy coverage; P1 and P2 may be reduced (future extension, not in MVP).

## 5. Strategy playbook — `testing-strategies.md` (MVP)

Skill-level file at `skills/subagent-driven-development/testing-strategies.md`. Five strategies in the MVP, all required for every P0 journey. New strategies are added by appending sections; no schema change is required — the canonical ID list is whatever sections exist in the file.

| ID | Meaning |
|---|---|
| `happy_path` | Journey runs end-to-end with valid inputs; final state matches `expected_outcome`. |
| `negative_path` | Invalid input, service failure, or timeout doesn't corrupt state or leak errors. |
| `state_persistence` | State survives page reload, navigation away and back, and logout-login mid-journey. |
| `feature_interaction` | Running an adjacent journey (sharing a tag in `feature_interactions`) immediately before or after doesn't break state. |
| `auth_boundary` | Protected steps reject unauthenticated or wrong-tenant actors. |

**Deferred to a later spec** (will be appended to the same file): `idempotency`, `concurrency`, `boundary_values`, `accessibility`, `visual_smoke`.

**Per-strategy section shape** (one `##` section per ID):
- `**When it applies:**`
- `**Legitimate N/A reasons:**`
- `**Example scenario shape:**`
- `**Common shallow mistakes:**`

**N/A rule**
A cell may be marked `na` with a justification, but only for these reasons:
- **Structurally inapplicable** (e.g., `auth_boundary` on a public-only journey).
- **Covered by another cell** (e.g., `happy_path` already asserts the property the strategy would check).
- **Explicitly deferred in the spec** (must cite the spec section that defers it).

"Too hard", "not critical", or one-word N/A is rejected by the testing-agent in Round 2/4.

## 6. Coverage matrix — shape and verification

Produced by solution-ITC negotiation, embedded in the signed solution-ITC YAML, consumed by the solution-runner.

```yaml
coverage_matrix:
  J1:
    happy_path:
      scenario:
        command: "npx playwright test tests/e2e/signup-flow.spec.ts --grep @J1-happy"
        assertion_shape: "end state: both users in workspace, roles=[owner,member], invite marked accepted in DB"
    negative_path:
      scenario:
        command: "npx playwright test tests/e2e/signup-flow.spec.ts --grep @J1-invalid-email"
        assertion_shape: "invalid email yields 400; no partial user record written"
    state_persistence:
      scenario:
        command: "npx playwright test tests/e2e/signup-flow.spec.ts --grep @J1-reload"
        assertion_shape: "after reload mid-invite, workspace state unchanged, user still authenticated"
    feature_interaction:
      scenario:
        command: "npx playwright test tests/e2e/signup-flow.spec.ts --grep @J1-after-J3"
        assertion_shape: "J1 executed immediately after J3 (password reset) still succeeds"
    auth_boundary:
      scenario:
        command: "npx playwright test tests/e2e/signup-flow.spec.ts --grep @J1-unauthorized"
        assertion_shape: "unauthenticated GET /workspaces/:id returns 401, not 200"
  J2:
    happy_path: { scenario: { command: "...", assertion_shape: "..." } }
    negative_path: { scenario: { command: "...", assertion_shape: "..." } }
    state_persistence: { na: "journey is a single atomic request; no multi-step state to persist" }
    feature_interaction: { scenario: { command: "...", assertion_shape: "..." } }
    auth_boundary: { scenario: { command: "...", assertion_shape: "..." } }
```

**Rules**
- Every journey listed in `journeys.yaml` must appear as a row.
- Every strategy ID currently in `testing-strategies.md` must appear as a column for every row.
- Every cell is either a `scenario` (with `command` and `assertion_shape`) or an `na` (with a non-empty justification string).
- Strategy column names are copied verbatim from the playbook file; the coding-agent and testing-agent must not invent new IDs. New strategies are introduced by editing `testing-strategies.md`, not by freelancing inside a matrix.
- `assertion_shape` is a required prose field describing the deep property the test asserts. Shallow shapes ("HTTP 200", "element visible") are rejected in Round 2/4 review.

## 7. Workflow integration — per-skill changes

### 7.1 `skills/brainstorming/SKILL.md`

**New sub-step in the design phase — Journey Enumeration**
After the feature design is approved and before writing the spec document, the agent:
1. Walks the PRD and the design's feature list, asking what end-to-end flows a user takes across features.
2. Produces draft entries for `<topic>-journeys.yaml` using the Section 4 schema.
3. Presents journeys with priorities to the user; obtains approval using the same gate pattern as the rest of the design.

**Write-design-doc step — extended**
Write both the design doc and `docs/superpowers/specs/<date>-<topic>-journeys.yaml`; commit both in the same commit.

**Spec self-review — additions**
- Every P0 journey has non-trivial steps and a specific `expected_outcome`.
- `feature_interactions` tags are drawn from the design's declared feature list (no invented tags).
- No journey is so vague that any strategy cell would be unwriteable.

### 7.2 `skills/writing-plans/SKILL.md`

**Plan schema — new fields**
- Each task gains an optional `contributes_to: [J1, J3]` field. Cited when the task clearly implements one or more journeys; omitted when unclear. Not a gate.
- Plan gains a top-level `## Journeys (reference)` section with a compact bulleted list of `{id, name}` pairs pulled from `journeys.yaml`. No detail duplication; journeys.yaml remains source of truth.

### 7.3 `skills/subagent-driven-development/SKILL.md`

**Solution-ITC negotiation — input expansion**
Before dispatching Round 1, the implementer loads `<topic>-journeys.yaml` and `skills/subagent-driven-development/testing-strategies.md` and passes both to the coding-agent.

**Vocabulary swap**
Every reference to "E2E scenarios list" is replaced with "coverage matrix".

**New post-negotiation gate**
Before dispatching the solution-runner, the implementer verifies the signed ITC's `coverage_matrix` has an entry for every journey in `journeys.yaml` and every strategy ID currently in `testing-strategies.md`. If not, the negotiation is rejected as incomplete and re-dispatched.

### 7.4 `skills/subagent-driven-development/coding-agent-prompt.md`

**Rewrite the "If Producing a Solution ITC" block**
- Required inputs: journeys.yaml, testing-strategies.md, scanned implementation.
- Output shape: replace `scenarios:` with `coverage_matrix:` using the Section 6 schema.
- Instructions: "use the canonical strategy IDs from the playbook file; do not invent new ones; every cell must be `scenario` or `na`; every `scenario` must have both `command` and `assertion_shape`."

### 7.5 `skills/subagent-driven-development/testing-agent-prompt.md`

**Rewrite the "If Reviewing a Solution ITC" block**
- Required reading: journeys.yaml plus testing-strategies.md, specifically the `Common shallow mistakes` and `Legitimate N/A reasons` sections per strategy.
- Checks:
  1. Every journey × every strategy has a non-empty decision.
  2. No `assertion_shape` is shallow. The prompt explicitly lists "HTTP 200 alone", "element visible alone", and "status code without DB or state verification" as rejectable.
  3. No `na` justification is lazy. The prompt lists legitimate N/A reasons from the playbook and rejects anything outside them.
  4. Every `scenario` command is runnable and tied to a real code path that the coding-agent has scanned.

### 7.6 `skills/subagent-driven-development/test-runner-solution-prompt.md`

**Input replacement**
Replace the "E2E Scenarios to Verify" input block with the full signed `coverage_matrix`.

**Execution rule**
Walk the matrix row-by-row, column-by-column. For each cell:
- If `scenario`: verify required services, run the command, capture output, classify PASS / FAIL / BLOCKED.
- If `na`: do not execute; echo the justification text into the report.

After the matrix, run the `full_suite` command once (unchanged).

**Report format**
Replace the flat `E2E Results:` block with a `matrix_results:` structure mirroring the input matrix shape. Example:

```yaml
matrix_results:
  J1:
    happy_path:       { status: PASS }
    negative_path:    { status: FAIL, failure_detail: "<stderr tail + failing test>", assertion_shape: "<echoed>" }
    state_persistence:{ status: PASS }
    feature_interaction: { status: PASS }
    auth_boundary:    { status: NA, justification: "journey is public-only" }
  J2: ...
full_suite: { status: PASS, summary: "142/142" }
overall: FAIL
```

**Overall verdict**
- **PASS** iff every `scenario` cell is PASS, every `na` cell has a non-empty justification echoed, and `full_suite` is PASS.
- **FAIL** iff any `scenario` cell is FAIL or any `na` cell echoes an empty justification.
- **BLOCKED** iff any required service check fails — report BLOCKED immediately without partial execution (same rule as today).

## 8. Feedback loop to implementer

Unchanged pattern: when `overall: FAIL`, the implementer dispatches a coding-agent to fix.

New signal: each failing cell's report includes the `assertion_shape` string echoed back, so the implementer sees not just which test failed but what deep property the test was asserting. Fixes can target intent, not just the failing line.

## 9. Files added, modified, and unchanged

**Added**
- `skills/subagent-driven-development/testing-strategies.md` (one-time, skill-level)
- `docs/superpowers/specs/<date>-<topic>-journeys.yaml` (per project, produced by brainstorm)

**Modified**
- `skills/brainstorming/SKILL.md`
- `skills/writing-plans/SKILL.md`
- `skills/subagent-driven-development/SKILL.md`
- `skills/subagent-driven-development/coding-agent-prompt.md` (solution-ITC block only)
- `skills/subagent-driven-development/testing-agent-prompt.md` (solution-ITC block only)
- `skills/subagent-driven-development/test-runner-solution-prompt.md`

**Unchanged**
- `skills/subagent-driven-development/implementer-prompt.md`
- `skills/subagent-driven-development/spec-reviewer-prompt.md`
- `skills/subagent-driven-development/code-quality-reviewer-prompt.md`
- `skills/subagent-driven-development/testing-agent-prompt.md` (task-ITC block)
- `skills/subagent-driven-development/coding-agent-prompt.md` (task-ITC block)
- `skills/subagent-driven-development/test-runner-task-prompt.md`

## 10. Risks and open questions

- **Matrix bloat for small projects.** A project with three journeys still writes 15 cells. Mitigation: `na` is a first-class outcome; structural inapplicability is normal and expected. If this proves too heavy in practice, priority-based column reduction (P1/P2 subsets) is the next lever.
- **Journey drift during implementation.** If a task uncovers a new journey not in `journeys.yaml`, the harness needs a path to add it. Initial rule: the implementer edits `journeys.yaml` and commits before proceeding to solution-ITC. Formalization deferred to a follow-up if drift proves common.
- **Strategy-playbook evolution.** New strategies added to `testing-strategies.md` retroactively apply to in-flight projects the next time solution-ITC runs. This is intentional — the matrix pulls from whatever IDs currently exist in the file — but it means "stable coverage for an in-flight branch" requires pinning the playbook version if strict reproducibility is needed. Not addressed in MVP.

## 11. Deferred to follow-up specs

- Incremental E2E execution (run coverage matrix per completed journey-group rather than only at end).
- Extended strategy playbook (idempotency, concurrency, boundary_values, accessibility, visual_smoke).
- Priority-based column reduction (P1/P2 journeys get a subset of strategies as a cost lever).
- Formal drift-handling workflow for journeys discovered during implementation.
- Issues 3 and 4 from `discussion.md` — main-agent context length and token cost — to be addressed by a separate brainstorm.
