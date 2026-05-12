# Inner-Loop Tiered Execution Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a 3-tier classification (`trivial` / `standard` / `heavy`) so each task in a plan runs only the gates it needs, with halt-and-escalate as the runtime safety net.

**Architecture:** All changes are documentation/prompt edits across two existing skills. The planner side (`skills/writing-plans/`) gains a tier rubric file plus instructions to emit an inline YAML `tier` block per task. The orchestrator side (`skills/subagent-driven-development/`) gains tier-aware dispatch routing, escalation parsing, and end-of-run ledger rendering. The implementer prompt gains a second exit path (`<ESCALATE>`).

**Tech Stack:** Markdown skill files, YAML schemas, no code. Verification is done via `grep` checks that expected markers are present in the updated files.

**Spec reference:** `docs/superpowers/specs/2026-04-25-inner-loop-tiered-execution-design.md`
**Journeys reference:** `docs/superpowers/specs/2026-04-25-inner-loop-tiered-execution-journeys.yaml`

---

## Journeys (reference)

- J1 — Plan with mixed tiers runs each task at its assigned tier
- J2 — Planner classifies a fresh plan using the rubric
- J3 — Planner defaults to heavy when no rubric clause matches
- J4 — Implementer escalates a misclassified trivial task
- J5 — Per-task escalation cap blocks a second escalation
- J6 — Heavy task signaling escalate halts the plan
- J7 — Task without a tier block is treated as heavy
- J8 — User overrides planner's tier before execution
- J9 — End-of-run escalation ledger is rendered
- J10 — Trivial task with malformed YAML block falls back to heavy

(Full detail lives in `docs/superpowers/specs/2026-04-25-inner-loop-tiered-execution-journeys.yaml`.)

---

## File Structure

| File | Action | Responsibility |
|---|---|---|
| `skills/writing-plans/tier-rubric.md` | **CREATE** | Authoritative rubric (`trivial` / `standard` / `heavy` clauses + examples). Source-of-truth document referenced from `writing-plans/SKILL.md`. |
| `skills/writing-plans/SKILL.md` | **MODIFY** | Reference the rubric, document the inline YAML `tier` block schema, instruct the planner to emit it per task and quote the matched clause. |
| `skills/subagent-driven-development/implementer-prompt.md` | **MODIFY** | Add `<ESCALATE>` exit path: structured block format, "no commit on escalate" rule, when to use vs. `BLOCKED`. |
| `skills/subagent-driven-development/SKILL.md` | **MODIFY** | Add tier-aware dispatch (read tier per task, run the matching flow), escalation handling protocol, end-of-run ledger rendering. |

No new code files. No new directories. No skills-cheatsheet update needed (this is a per-task feature, not a workflow people need to memorize).

---

## Task sequencing

Tasks 1 and 3 are independent. Task 2 depends on Task 1 (writing-plans SKILL.md cites the rubric file). Task 4 depends on Task 3 (orchestrator dispatch references the `<ESCALATE>` contract documented in the implementer prompt). Task 5 depends on Task 4 (escalation handling lives in the same SKILL.md as dispatch, builds on its vocabulary).

Recommended order: **1 → 2 → 3 → 4 → 5**.

---

### Task 1: Create the tier rubric file

**Files:**
- Create: `skills/writing-plans/tier-rubric.md`

**Contributes to:** [J2, J3]

- [ ] **Step 1: Write the failing check**

```bash
test -f skills/writing-plans/tier-rubric.md && \
  grep -q '^## `trivial`' skills/writing-plans/tier-rubric.md && \
  grep -q '^## `standard`' skills/writing-plans/tier-rubric.md && \
  grep -q '^## `heavy`' skills/writing-plans/tier-rubric.md && \
  grep -q '^- \*\*T1\.\*\*' skills/writing-plans/tier-rubric.md && \
  grep -q '^- \*\*S4\.\*\*' skills/writing-plans/tier-rubric.md && \
  grep -q '^- \*\*H5\.\*\*' skills/writing-plans/tier-rubric.md && \
  echo OK
```

Expected: nothing prints (file does not exist yet).

- [ ] **Step 2: Create the rubric file with this exact content**

```markdown
# Tier Rubric

Used by the writing-plans skill to classify each task in a plan into one of three execution tiers. Tier definitions live in the inner-loop tiered execution design (`docs/superpowers/specs/2026-04-25-inner-loop-tiered-execution-design.md`).

**How to apply this rubric:** for each task, walk the clauses top-down (`trivial` → `standard` → `heavy`) and pick the first tier whose match conditions are satisfied. Quote the matched clause id (e.g., `T1`, `S2`, `H4`) in the task's `tier_reason` field. If no clause matches cleanly, default to `heavy` with `tier_reason` citing `H5`.

**Default-heavy rule:** when in doubt, classify as `heavy`. Misclassifying a heavy task as trivial costs more than the reverse — implementer can escalate up, but cannot escalate down.

## `trivial`

**Runs:** implementer agent only.

Match if **all** of the following are true:

- **T1.** No production source code is touched (only `*.md`, `.gitignore`, `LICENSE`, `.editorconfig`, `CODEOWNERS`, lockfile-only dep bumps, or similar config-only files).
- **T2.** No new files of code are added.
- **T3.** No behavior change.
- **T4.** No new tests are required, and no existing test is expected to break.

**Examples:** README edit, `.gitignore` entry, typo fix, license header, package version bump without code change, CODEOWNERS update.

## `standard`

**Runs:** implementer → code-reviewer → test-runner (existing tests only).

Match if **all** of the following are true and `trivial` did not match:

- **S1.** Change is behavior-preserving (rename, extract, inline, move between files, format, dead-code removal).
- **S2.** No new public API surface (no new exported functions/types/endpoints/CLI flags/env vars).
- **S3.** No new tests needed (existing tests cover the change, or change is test-agnostic).
- **S4.** Test-only task — adds or updates tests for existing code without changing the code under test (qualifies as `standard` even though it adds test files).

**Examples:** symbol rename across files, extract helper, inline one-use function, delete unreferenced code, reorganize imports, dep bump whose API didn't change, adding tests for an existing function.

## `heavy`

**Runs:** ITC negotiation (coding + testing agents) → implementer → spec-reviewer → code-reviewer → test-runner.

Match if **any** of the following is true:

- **H1.** Adds new behavior (new feature, new branch of logic, new side effect).
- **H2.** Adds or changes a public API.
- **H3.** Needs new tests (other than the test-only S4 case).
- **H4.** Changes a contract at a module boundary (input/output shape, error semantics).
- **H5.** None of the above matched cleanly — default to `heavy`.

**Examples:** add a new endpoint, add a new CLI flag, change a DB schema, introduce a new service, refactor that also fixes a bug (behavior change sneaking in).
```

- [ ] **Step 3: Run the check to verify it passes**

```bash
test -f skills/writing-plans/tier-rubric.md && \
  grep -q '^## `trivial`' skills/writing-plans/tier-rubric.md && \
  grep -q '^## `standard`' skills/writing-plans/tier-rubric.md && \
  grep -q '^## `heavy`' skills/writing-plans/tier-rubric.md && \
  grep -q '^- \*\*T1\.\*\*' skills/writing-plans/tier-rubric.md && \
  grep -q '^- \*\*S4\.\*\*' skills/writing-plans/tier-rubric.md && \
  grep -q '^- \*\*H5\.\*\*' skills/writing-plans/tier-rubric.md && \
  echo OK
```

Expected: `OK` printed.

- [ ] **Step 4: Commit**

```bash
git add skills/writing-plans/tier-rubric.md
git commit -m "Add tier rubric for inner-loop classification"
```

---

### Task 2: Wire the tier rubric into writing-plans SKILL.md

**Files:**
- Modify: `skills/writing-plans/SKILL.md`

**Contributes to:** [J2, J3]

This task adds two things to the SKILL: (a) instructions for the planner to emit a tier YAML block per task, and (b) a reference to `tier-rubric.md` for classification.

- [ ] **Step 1: Write the failing check**

```bash
grep -q '^## Tier Classification' skills/writing-plans/SKILL.md && \
  grep -q 'tier-rubric.md' skills/writing-plans/SKILL.md && \
  grep -q 'tier_reason' skills/writing-plans/SKILL.md && \
  grep -q '^tier: trivial' skills/writing-plans/SKILL.md && \
  echo OK
```

Expected: nothing prints (markers not yet added).

- [ ] **Step 2: Edit `skills/writing-plans/SKILL.md` — insert a new section between "Bite-Sized Task Granularity" and "Plan Document Header"**

Insert this section so the planner sees tier classification rules before it reads about plan document layout:

`````markdown
## Tier Classification

Every task in the plan MUST be classified into one of three execution tiers — `trivial`, `standard`, or `heavy` — using the rubric in `./tier-rubric.md`. The tier determines which inner-loop gates run (ITC negotiation, spec-reviewer, code-reviewer, test-runner). See `docs/superpowers/specs/2026-04-25-inner-loop-tiered-execution-design.md` §2 for what runs at each tier.

**How to classify:**
1. Read `./tier-rubric.md`.
2. For each task, walk the rubric top-down. Pick the first tier whose clauses are satisfied.
3. Quote the matched clause id in `tier_reason` (e.g., `"T1: only modifies .gitignore (config-only file); T2/T3/T4 satisfied"`).
4. If no clause matches cleanly, classify as `heavy` and cite `H5` in `tier_reason`.

**Required schema — inline YAML block immediately after each task header:**

````markdown
### Task N: [Component Name]

```yaml
tier: trivial
tier_reason: "T1: only modifies .gitignore (config-only file); T2/T3/T4 satisfied"
```

**Files:**
- ...
````

**Required fields:**
- `tier` — one of `trivial | standard | heavy`
- `tier_reason` — string quoting the matched rubric clause id and a short explanation

**Default-heavy rule:** when uncertain, choose `heavy`. Misclassifying down (e.g., labeling a heavy task `trivial`) shortcuts safety gates; misclassifying up only costs extra agent work. Bias toward safety.

**Test-only tasks** (e.g., "add tests for existing function X") qualify as `standard` via clause `S4`, not `heavy`.

**Trivial example:**

```yaml
tier: trivial
tier_reason: "T1: only modifies README.md (config-only file); T2/T3/T4 satisfied"
```

**Standard example:**

```yaml
tier: standard
tier_reason: "S1: rename of internal helper across 3 files, behavior-preserving; S2/S3 satisfied"
```

**Heavy example:**

```yaml
tier: heavy
tier_reason: "H2: adds a new exported function buildIndex() to public API"
```
`````

- [ ] **Step 3: Update the `## Task Structure` example to include a tier block**

Find the existing `## Task Structure` section in `skills/writing-plans/SKILL.md`. The current first lines are:

```markdown
### Task N: [Component Name]

**Files:**
```

Replace those three lines with:

````markdown
### Task N: [Component Name]

```yaml
tier: heavy
tier_reason: "H1: adds new behavior — implements feature X end-to-end"
```

**Files:**
````

- [ ] **Step 4: Update `## Self-Review` to add a tier-coverage check**

Find the `## Self-Review` section. After the existing numbered checks (Spec coverage, Placeholder scan, Type consistency), add a fourth check:

```markdown
**4. Tier coverage:** Every task has a `tier` block with `tier` and `tier_reason`. Every `tier_reason` quotes a clause id from `./tier-rubric.md` (e.g., `T1`, `S2`, `H4`). Tasks with no clear match are tagged `heavy` citing `H5`.
```

- [ ] **Step 5: Run the check to verify it passes**

```bash
grep -q '^## Tier Classification' skills/writing-plans/SKILL.md && \
  grep -q 'tier-rubric.md' skills/writing-plans/SKILL.md && \
  grep -q 'tier_reason' skills/writing-plans/SKILL.md && \
  grep -q '^tier: trivial' skills/writing-plans/SKILL.md && \
  echo OK
```

Expected: `OK` printed.

- [ ] **Step 6: Commit**

```bash
git add skills/writing-plans/SKILL.md
git commit -m "Wire tier classification into writing-plans SKILL"
```

---

### Task 3: Add escalation exit path to implementer prompt

**Files:**
- Modify: `skills/subagent-driven-development/implementer-prompt.md`

**Contributes to:** [J4, J5, J6]

This task documents the `<ESCALATE>` exit path on the implementer subagent. The orchestrator-side handling lives in Task 5; this task only defines the *contract* the implementer follows.

- [ ] **Step 1: Write the failing check**

```bash
grep -q '<ESCALATE>' skills/subagent-driven-development/implementer-prompt.md && \
  grep -q 'requested_tier' skills/subagent-driven-development/implementer-prompt.md && \
  grep -q 'rubric_clause_violated' skills/subagent-driven-development/implementer-prompt.md && \
  grep -q 'attempted_before_halt' skills/subagent-driven-development/implementer-prompt.md && \
  grep -q 'never ship partial work' skills/subagent-driven-development/implementer-prompt.md && \
  echo OK
```

Expected: nothing prints.

- [ ] **Step 2: Edit `skills/subagent-driven-development/implementer-prompt.md` — add escalation section**

Find the `## When You're in Over Your Head` section. After its existing content (ending at the line `The controller can provide more context, re-dispatch with a more capable model, or break the task into smaller pieces.`), add this new section before `## Before Reporting Back: Self-Review`:

````markdown
    ## Tier Escalation (`<ESCALATE>`)

    Each task is dispatched at a tier (`trivial`, `standard`, or `heavy`). If the
    task description and acceptance criteria turn out to require gates the
    current tier doesn't run, **halt before committing any code** and signal
    escalation. Escalations never ship partial work.

    **When to escalate:**
    - You're at `trivial` and the task requires adding a code file, changing
      behavior, or adding tests.
    - You're at `standard` and the task requires a new public API, new
      behavior, or contract change at a module boundary.
    - You're at `heavy` — escalation is not available; report `BLOCKED` instead.

    **What "halt before committing" means:**
    - Do not run `git commit`.
    - Discard any uncommitted edits in your worktree (e.g., via `git restore .`
      or `git stash drop` after `git stash`). The re-dispatched run starts fresh.

    **Escalation report format** — return this YAML block as the FIRST content of
    your reply (before any other text):

    ```
    <ESCALATE>
    task_id: T<N>                    # task id from the plan
    current_tier: trivial            # the tier you were dispatched at
    requested_tier: standard         # MUST be strictly higher than current_tier
    reason: >
      One-paragraph explanation: what about the task requires a higher tier?
    rubric_clause_violated: "trivial: no new files of code (T2)"
    evidence:
      - "Concrete observation 1 (e.g., 'AC 3 requires new file scripts/foo.sh')"
      - "Concrete observation 2 (optional)"
    attempted_before_halt: false     # true if any code was written then discarded
    </ESCALATE>
    ```

    **Escalation rules:**
    - `requested_tier` MUST be strictly higher than `current_tier` (one-way only).
    - You may escalate at most ONCE per task; the orchestrator enforces this.
    - `heavy` cannot escalate. If you're stuck at `heavy`, report `BLOCKED`.

    **Escalate vs. BLOCKED:**
    - **Escalate** — task is well-specified but its assigned tier is too light.
    - **BLOCKED** — task itself is unclear, contradictory, requires architectural
      decisions, or you cannot proceed regardless of tier.
````

- [ ] **Step 3: Update the `**Status:**` enum line in the Report Format section**

Find this line in `## Report Format`:

```
    - **Status:** DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT
```

Replace with:

```
    - **Status:** DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT | ESCALATE
```

And immediately after the existing line that reads `Use NEEDS_CONTEXT if you need information that wasn't provided.` (within the closing prose), add this sentence:

```
Use ESCALATE (with the structured `<ESCALATE>` block defined above) when the task requires gates a higher tier provides.
```

- [ ] **Step 4: Run the check to verify it passes**

```bash
grep -q '<ESCALATE>' skills/subagent-driven-development/implementer-prompt.md && \
  grep -q 'requested_tier' skills/subagent-driven-development/implementer-prompt.md && \
  grep -q 'rubric_clause_violated' skills/subagent-driven-development/implementer-prompt.md && \
  grep -q 'attempted_before_halt' skills/subagent-driven-development/implementer-prompt.md && \
  grep -q 'never ship partial work' skills/subagent-driven-development/implementer-prompt.md && \
  echo OK
```

Expected: `OK` printed.

- [ ] **Step 5: Commit**

```bash
git add skills/subagent-driven-development/implementer-prompt.md
git commit -m "Add tier escalation exit path to implementer prompt"
```

---

### Task 4: Add tier-aware dispatch to subagent-driven-development SKILL.md

**Files:**
- Modify: `skills/subagent-driven-development/SKILL.md`

**Contributes to:** [J1, J7, J8, J10]

This task teaches the orchestrator to read each task's `tier` block and run only the matching gates. Escalation handling and ledger come in Task 5.

- [ ] **Step 1: Write the failing check**

```bash
grep -q '^## Tier-Aware Dispatch' skills/subagent-driven-development/SKILL.md && \
  grep -q 'fail-safe.*heavy' skills/subagent-driven-development/SKILL.md && \
  grep -q '^### Trivial flow' skills/subagent-driven-development/SKILL.md && \
  grep -q '^### Standard flow' skills/subagent-driven-development/SKILL.md && \
  grep -q '^### Heavy flow' skills/subagent-driven-development/SKILL.md && \
  echo OK
```

Expected: nothing prints.

- [ ] **Step 2: Edit `skills/subagent-driven-development/SKILL.md` — add `Tier-Aware Dispatch` section**

Insert this section immediately before the existing `## ITC Negotiation` section (so the orchestrator learns about tiers before reading the heavy-flow ITC details that may now be skipped):

````markdown
## Tier-Aware Dispatch

Each task in a plan carries an inline YAML `tier` block (see `skills/writing-plans/SKILL.md` and `skills/writing-plans/tier-rubric.md`). The orchestrator MUST read the tier per task and run only the gates that tier requires.

### Reading the tier

For each task, parse the YAML block immediately following the task header:

```yaml
tier: <trivial | standard | heavy>
tier_reason: "<rubric clause id and explanation>"
```

**Fail-safe rules — when in doubt, treat the task as `heavy`:**
- No tier block present → treat as `heavy`. Log a warning naming the task.
- YAML parse error → treat as `heavy`. Log a warning with the parse error and task id.
- `tier` value is not one of `trivial | standard | heavy` → treat as `heavy`. Log a warning naming the task and the unknown value.
- The user may hand-edit `tier` in `plan.md`; the orchestrator trusts the file as the source of truth at execution time and does NOT re-validate against the rubric.

### Per-tier flows

The flows below replace the per-task block of the main process flowchart for every task. The flowchart's solution-level negotiation, E2E + full-suite test-runner, and final code reviewer (after all tasks complete) are unchanged.

#### Trivial flow

1. Dispatch implementer subagent (`./implementer-prompt.md`) with task spec + scene-setting context.
2. Handle implementer status (`DONE`, `DONE_WITH_CONCERNS`, `BLOCKED`, `NEEDS_CONTEXT`, `ESCALATE`) per the existing `## Handling Implementer Status` and Task-5 `## Escalation Handling` rules.
3. On `DONE`: mark task complete in TodoWrite. Skip ITC negotiation, spec-reviewer, code-reviewer, and test-runner — none apply at trivial.

#### Standard flow

1. Dispatch implementer subagent (`./implementer-prompt.md`).
2. On `DONE`: dispatch code quality reviewer subagent (`./code-quality-reviewer-prompt.md`).
3. On reviewer ✅: dispatch unit test-runner (`./test-runner-task-prompt.md`) restricted to existing tests (do not require new tests for the change).
4. On test-runner `PASS`: mark task complete.
5. Skip ITC negotiation and spec-reviewer — neither applies at standard.

#### Heavy flow

Run the full per-task block exactly as the main process flowchart describes:

1. ITC negotiation (coding-agent ↔ testing-agent, up to 5 rounds, write contract file).
2. Implementer subagent.
3. Spec-reviewer subagent.
4. Code quality reviewer subagent.
5. Unit test-runner; integration test-runner if `tiers_required` includes `integration`.

### Dispatch decision (pseudocode)

```
for task in plan.tasks:
    tier = parse_tier_block(task) or "heavy"   # fail-safe
    match tier:
        case "trivial": run_trivial_flow(task)
        case "standard": run_standard_flow(task)
        case "heavy":    run_heavy_flow(task)
```
````

- [ ] **Step 3: Run the check to verify it passes**

```bash
grep -q '^## Tier-Aware Dispatch' skills/subagent-driven-development/SKILL.md && \
  grep -q 'fail-safe.*heavy' skills/subagent-driven-development/SKILL.md && \
  grep -q '^### Trivial flow' skills/subagent-driven-development/SKILL.md && \
  grep -q '^### Standard flow' skills/subagent-driven-development/SKILL.md && \
  grep -q '^### Heavy flow' skills/subagent-driven-development/SKILL.md && \
  echo OK
```

Expected: `OK` printed.

- [ ] **Step 4: Commit**

```bash
git add skills/subagent-driven-development/SKILL.md
git commit -m "Add tier-aware dispatch to subagent-driven-development"
```

---

### Task 5: Add escalation handling and end-of-run ledger to subagent-driven-development SKILL.md

**Files:**
- Modify: `skills/subagent-driven-development/SKILL.md`

**Contributes to:** [J4, J5, J6, J9]

This task adds the orchestrator-side escalation protocol (parsing `<ESCALATE>`, validating, updating `plan.md`, fresh re-dispatch) and the end-of-run escalation ledger.

- [ ] **Step 1: Write the failing check**

```bash
grep -q '^## Escalation Handling' skills/subagent-driven-development/SKILL.md && \
  grep -q 'fresh re-dispatch' skills/subagent-driven-development/SKILL.md && \
  grep -q 'one-way only' skills/subagent-driven-development/SKILL.md && \
  grep -q 'escalation_count' skills/subagent-driven-development/SKILL.md && \
  grep -q '## Escalations' skills/subagent-driven-development/SKILL.md && \
  echo OK
```

Expected: nothing prints (markers not yet added beyond Task 4's content).

- [ ] **Step 2: Edit `skills/subagent-driven-development/SKILL.md` — add `Escalation Handling` section**

Insert this section immediately after the new `## Tier-Aware Dispatch` section (added in Task 4) and before `## ITC Negotiation`:

````markdown
## Escalation Handling

Implementer subagents may exit with `ESCALATE` (see `./implementer-prompt.md`). The escalation contract: implementer halts before committing, discards any partial work, and returns a structured `<ESCALATE>` block as the first content of its reply.

### Detecting an escalation

After the implementer returns, check whether the reply begins with `<ESCALATE>`:

- If yes → handle per this section. Do NOT run any further gates for this dispatch attempt.
- If no → process the reply per `## Handling Implementer Status` (DONE, DONE_WITH_CONCERNS, BLOCKED, NEEDS_CONTEXT).

### Validating the escalation

Parse the YAML inside the `<ESCALATE>` block. Reject the escalation (and halt the plan with a clear user message) if any of the following is true:

- `requested_tier` is not strictly higher than `current_tier` (one-way only — never demote).
- `current_tier` is `heavy` (heavy is the top tier; nothing higher exists).
- The task's `escalation_count` is already `1` (per-task cap reached).

When rejecting because `current_tier` is `heavy` or because the cap is reached, surface a user-facing message naming the task id and the reason, then halt.

### Re-dispatching at the new tier

If validation passes:

1. Update `plan.md` for the task (audit trail):
   - `tier` ← `requested_tier`
   - `tier_reason` ← `"escalated from <current_tier>: <reason>"`
   - `escalation_count` ← previous + 1 (start at 0; absent treated as 0)
2. Append the escalation note (full `<ESCALATE>` payload) to the in-memory escalation ledger for the run.
3. **Fresh re-dispatch at the new tier — same call as a first dispatch.** Do NOT inject the escalation note into the new flow's prompt. The task description in `plan.md` is the source of truth; the escalation note is audit-only.

### End-of-run escalation ledger

After the last task in the plan completes (or the plan halts), append the ledger to `plan.md` if any escalations occurred:

```markdown
## Escalations (<count>)

- T<id>: <original_tier> → <new_tier>. Reason: <reason>. Rubric clause violated: <clause id>.
- T<id>: <original_tier> → <new_tier>. Reason: <reason>. Rubric clause violated: <clause id>.
```

If no escalations occurred during the run, do not append the section.

### Caps and guardrails

| Rule | Value |
|---|---|
| Max escalations per task | 1 (orchestrator enforces) |
| Plan-wide escalation budget | none (per-task cap is the only cap) |
| `heavy` may not escalate | enforced — heavy escalation halts the plan |
| Demotion at runtime | forbidden — escalation is one-way only |
| Partial work on escalate | discarded by implementer; orchestrator does NOT inject the escalation note into the re-dispatched prompt |
````

- [ ] **Step 3: Update `## Handling Implementer Status` to mention `ESCALATE`**

Find the `## Handling Implementer Status` section (it starts with the line `Implementer subagents report one of four statuses. Handle each appropriately:`). Replace `four statuses` with `five statuses`. Then, immediately before the `**Never** ignore an escalation or force the same model to retry without changes.` line, insert this status block:

```markdown
**ESCALATE:** The implementer's reply begins with an `<ESCALATE>` YAML block. Process per `## Escalation Handling`. Do NOT proceed to spec or quality review for this dispatch attempt.

```

- [ ] **Step 4: Update `## Red Flags` `**Never:**` list to forbid escalation note injection**

Find the `**Never:**` list under `## Red Flags`. Add these two bullets at the end of the list (after the existing last bullet about forgetting to commit the ITC):

```markdown
- Inject the implementer's escalation note into the re-dispatched flow's prompt — escalation notes are audit-only; the next agent reads the task description, not the note
- Demote a task's tier at runtime — escalation is one-way only
```

- [ ] **Step 5: Run the check to verify it passes**

```bash
grep -q '^## Escalation Handling' skills/subagent-driven-development/SKILL.md && \
  grep -q 'fresh re-dispatch' skills/subagent-driven-development/SKILL.md && \
  grep -q 'one-way only' skills/subagent-driven-development/SKILL.md && \
  grep -q 'escalation_count' skills/subagent-driven-development/SKILL.md && \
  grep -q '## Escalations' skills/subagent-driven-development/SKILL.md && \
  grep -q 'five statuses' skills/subagent-driven-development/SKILL.md && \
  grep -q 'ESCALATE:' skills/subagent-driven-development/SKILL.md && \
  echo OK
```

Expected: `OK` printed.

- [ ] **Step 6: Commit**

```bash
git add skills/subagent-driven-development/SKILL.md
git commit -m "Add escalation handling and end-of-run ledger to subagent-driven-development"
```

---

## Post-implementation manual verification (optional)

These checks exercise the integration end-to-end. They are not blocking for plan completion (they require running the live skills against a fixture), but they are the cleanest way to confirm the design works.

1. **J2 (planner emits tier blocks):** Run `writing-plans` on a small spec containing a docs-only task, a rename task, and a new-feature task. Verify `plan.md` contains a `tier` block per task with `trivial`, `standard`, and `heavy` respectively, and each `tier_reason` quotes a valid clause id from `tier-rubric.md`.

2. **J1 (mixed-tier execution):** Run `subagent-driven-development` on the plan from step 1. Verify the dispatch log shows: implementer-only for the trivial task; implementer + code-reviewer + test-runner for the standard task; full ITC + spec-reviewer + code-reviewer + test-runner for the heavy task.

3. **J4 (escalation):** Hand-edit a `plan.md` to mistag a code-adding task as `tier: trivial`. Run subagent-driven-development. Verify the implementer halts with `<ESCALATE>`, the orchestrator updates `plan.md` to `tier: standard` with `escalated from trivial: …` reason, and the standard flow runs to completion. Verify the end-of-run ledger appears.

4. **J7 (missing tier block):** Hand-edit `plan.md` to remove a tier block from one task. Run subagent-driven-development. Verify the orchestrator logs a warning and runs that task at heavy.
