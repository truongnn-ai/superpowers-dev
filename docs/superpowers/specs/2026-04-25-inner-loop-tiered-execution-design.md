# Inner-Loop Tiered Execution

**Date:** 2026-04-25
**Status:** Draft for review
**Owner:** truong.nguyen@codelink.io

## 1. Problem & Goal

**Problem.** The superpowers inner loop (ITC negotiation → implementer → spec-reviewer → code-reviewer → test-runner) runs uniformly for every task in a plan. Trivial tasks (README edits, `.gitignore` changes, version bumps) pay the full cost — multiple agent runs, negotiation overhead, review passes — while producing no safety value. This drives high token cost and wall-clock latency on work that doesn't need it.

**Goal.** Introduce a 3-tier classification of tasks (`trivial` / `standard` / `heavy`) so each task runs only the gates it needs. Classification happens **at plan-writing time**, is recorded in `plan.md`, and is reviewable before execution.

**Non-goals.**
- PRD-level or brainstorm-level triage (origin spec workflow is unchanged).
- Runtime de-escalation (a task's tier can only go up, never down, during execution).
- Cross-task budgets spanning multiple tasks beyond the existing plan structure.

## 2. Tier definitions

Steps listed in processing order.

| Step | `trivial` | `standard` | `heavy` |
|---|---|---|---|
| ITC negotiation (coding + testing agents) | ❌ | ❌ | ✅ |
| Implementer agent | ✅ | ✅ | ✅ |
| Spec-reviewer agent | ❌ | ❌ | ✅ |
| Code-reviewer agent | ❌ | ✅ | ✅ |
| Test-runner agent | ❌ | ✅ (existing tests only) | ✅ |

Cost intuition (relative to `heavy = 1.0`):
- `trivial` ≈ 0.10–0.15 (single agent call)
- `standard` ≈ 0.40–0.50 (skips the expensive ITC negotiation + spec-review pair)
- `heavy` = 1.0 (full current loop)

**Why these splits.** The dominant cost in the current loop is the ITC negotiation + spec-review pair (multi-agent negotiation, multi-turn). `standard` removes those for behavior-preserving changes while keeping the cheap safety nets (code review, test runner on existing tests). `trivial` removes everything except the implementer for changes that don't touch code at all.

## 3. Classification (rubric + mandatory evidence)

The planner applies the rubric below when writing each task. It **must** quote the matched clause in `tier_reason`. If no clause matches, the task defaults to `heavy`.

### `trivial` — implementer agent only

Match if **all** of the following are true:

- **T1.** No production source code is touched (only `*.md`, `.gitignore`, `LICENSE`, `.editorconfig`, `CODEOWNERS`, lockfile-only dep bumps, or similar config-only files).
- **T2.** No new files of code are added.
- **T3.** No behavior change.
- **T4.** No new tests are required, and no existing test is expected to break.

Examples: README edit, `.gitignore` entry, typo fix, license header, package version bump without code change, CODEOWNERS update.

### `standard` — implementer → code-reviewer → test-runner (existing tests only)

Match if **all** of the following are true and `trivial` did not match:

- **S1.** Change is behavior-preserving (rename, extract, inline, move between files, format, dead-code removal).
- **S2.** No new public API surface (no new exported functions/types/endpoints/CLI flags/env vars).
- **S3.** No new tests needed (existing tests cover the change, or change is test-agnostic).
- **S4.** Test-only task — adds or updates tests for existing code without changing the code under test (qualifies as `standard` even though it adds test files).

Examples: symbol rename across files, extract helper, inline one-use function, delete unreferenced code, reorganize imports, dep bump whose API didn't change, adding tests for existing function.

### `heavy` — full loop (ITC → implementer → spec-reviewer → code-reviewer → test-runner)

Match if **any** of the following is true:

- **H1.** Adds new behavior (new feature, new branch of logic, new side effect).
- **H2.** Adds or changes a public API.
- **H3.** Needs new tests (other than the test-only S4 case).
- **H4.** Changes a contract at a module boundary (input/output shape, error semantics).
- **H5.** None of the above matched cleanly — default to `heavy`.

Examples: add a new endpoint, add a new CLI flag, change a DB schema, introduce a new service, refactor that also fixes a bug (behavior change sneaking in).

## 4. Plan schema

Each task in `plan.md` carries an inline YAML block immediately after its header:

````markdown
## T7: Add `.gitignore` entry for .env.local

```yaml
tier: trivial
tier_reason: "T1: only modifies .gitignore (config-only file); T2/T3/T4 satisfied"
```

**Description:** ...
**Acceptance criteria:** ...
````

**Required fields:** `tier` (one of `trivial | standard | heavy`), `tier_reason` (string quoting the matched rubric clause).

**Parsing rules:**
- Absence of a tier block → treat as `heavy` (fail-safe).
- Malformed YAML → treat as `heavy`, log a warning.
- Unknown `tier` value → treat as `heavy`, log a warning.

**User override:** users may hand-edit `tier` and `tier_reason` in `plan.md` before execution. The orchestrator trusts the file at execution time and does not re-validate against the rubric.

## 5. Escalation protocol

Implementer subagents have two exit paths: `done` and `escalate`.

### When implementer escalates

1. **Stops before committing any code.** Any uncommitted changes in its worktree are discarded. Escalations never ship partial work.
2. **Returns a structured `<ESCALATE>` block** as the first content of its return message:

```yaml
<ESCALATE>
task_id: T7
current_tier: trivial
requested_tier: standard        # must be strictly higher than current_tier
reason: >
  Task tagged trivial but requires adding scripts/preflight.sh — this is a
  new code file, violating clause T2.
rubric_clause_violated: "trivial: no new files of code (T2)"
evidence:
  - "New file scripts/preflight.sh needed per AC 3"
  - "CI workflow change implied by the new entry"
attempted_before_halt: false
</ESCALATE>
```

### Orchestrator behavior on escalation

```
on_implementer_return(result, task):
    if result.starts_with("<ESCALATE>"):
        note = parse_escalation(result)
        validate(note.requested_tier > task.current_tier)   # strictly higher
        validate(task.escalation_count < 1)                 # per-task cap
        validate(task.current_tier != "heavy")              # heavy cannot escalate

        # Update plan.md (audit trail, NOT context for next agent)
        task.tier             = note.requested_tier
        task.tier_reason      = f"escalated from {note.current_tier}: {note.reason}"
        task.escalation_count = 1
        append_to(escalation_ledger, note)

        # Fresh re-dispatch at new tier — same call as a first dispatch
        dispatch_tier_flow(task)
    else:
        proceed_with_remaining_gates(task)
```

### Caps and guardrails

| Rule | Value | Why |
|---|---|---|
| Max escalations per task | 1 | A second escalation indicates a fundamentally bad task spec. |
| Plan-wide escalation budget | none | The per-task cap already guards individual tasks; planner-level systemic issues are surfaced via the end-of-run ledger instead. |
| `heavy` may not escalate | enforced | `heavy` is the top tier; a heavy implementer signaling escalate halts the plan. |
| Demotion at runtime | forbidden | One-way escalation prevents an implementer from talking a task down into a cheaper tier. |
| Partial work on escalate | discarded | Re-dispatch is fresh; escalation note is audit-only and is not injected into the next agent's prompt. |

### End-of-run ledger

After the plan completes (or halts), the orchestrator appends an escalation ledger to `plan.md`:

```markdown
## Escalations (2)

- T7: trivial → standard. Reason: task required new script file. Rubric clause violated: T2.
- T11: standard → heavy. Reason: rename introduced a new public API. Rubric clause violated: S2.
```

This ledger is the feedback loop. Repeated patterns indicate the rubric or planner prompt needs updating.

## 6. Integration

**`skills/writing-plans/`** gains:
- A new section containing the rubric in §3 verbatim.
- Instruction to the planner: for every task, emit the inline YAML block in §4 and quote the matched clause in `tier_reason`.
- Default-heavy behavior when no clause matches.

**`skills/subagent-driven-development/`** gains:
- A tier-aware dispatcher that reads the YAML block per task and selects the flow from §2.
- Escalation protocol handling per §5 (parsing `<ESCALATE>`, validating, updating `plan.md`, re-dispatching).
- End-of-run ledger rendering.

**Implementer subagent prompt** gains:
- Two exit paths (`done` / `escalate`).
- The exact `<ESCALATE>` block format and the rule that no code may be committed when escalating.

**No new skills, no shared submodules.** Changes are confined to the two existing skills plus the implementer subagent prompt.

## 7. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Planner over-assigns `heavy` — no savings | Rubric is explicit and prioritized top-down; escalation ledger surfaces drift over time so the rubric can be tightened. |
| Planner over-assigns `trivial` — quality regressions | Implementer can halt+escalate; rubric defaults to `heavy` on doubt (H5). |
| Users misunderstand the tier in `plan.md` | `tier_reason` quotes the matched clause, making intent auditable without needing to consult the rubric. |
| Escalation loop (weird prompts make implementer keep escalating) | Per-task cap = 1; `heavy` cannot escalate; second escalation halts the plan. |
| Spec-reviewer is bypassed on `standard` and bugs sneak in | Accepted: `standard` is behavior-preserving by definition (S1), so a spec to review wouldn't meaningfully exist. Code-reviewer remains as a backstop. |
| Hand-edited `plan.md` introduces a malformed tier block | Parser treats malformed YAML as `heavy` (fail-safe) and logs a warning. |

## 8. Success criteria

1. A plan of mixed-tier tasks runs end-to-end with the right gates per task.
2. `plan.md` after a run clearly shows each task's tier, reason, and any escalations.
3. On a realistic plan (e.g., 10 tasks with 3 trivial / 4 standard / 3 heavy), aggregate token cost drops by roughly **35–50%** vs. the current uniform-heavy flow — measured once in a follow-up evaluation, not blocked on exact numbers.
4. No new skill files created; changes confined to `writing-plans`, `subagent-driven-development`, and the implementer subagent prompt.

## 9. Feature list

Tags used by the journeys (`docs/superpowers/specs/2026-04-25-inner-loop-tiered-execution-journeys.yaml`):

- `rubric-classification` — planner applies §3 rubric to assign a tier and quote evidence.
- `plan-schema` — inline YAML tier block per task in `plan.md`; parsing rules including fail-safe to `heavy`.
- `tier-dispatch` — orchestrator reads tier and runs the matching flow from §2.
- `escalation-protocol` — `<ESCALATE>` block, per-task cap, fresh re-dispatch, heavy-cannot-escalate.
- `escalation-ledger` — end-of-run summary appended to `plan.md`.
- `default-heavy-fallback` — missing tier, malformed block, unknown value, or no rubric clause matched all route to `heavy`.
