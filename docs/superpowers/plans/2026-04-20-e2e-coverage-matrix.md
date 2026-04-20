# E2E Coverage Matrix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace free-form solution-ITC scenarios with a mechanically-enforced Journey × Strategy coverage matrix by adding a skill-level testing-strategies playbook, a spec-time journeys.yaml artifact, and matrix-aware prompts for the coding-agent, testing-agent, and solution-runner.

**Architecture:** Two new artifacts (`skills/subagent-driven-development/testing-strategies.md` once; `docs/superpowers/specs/<date>-<topic>-journeys.yaml` per project) and six prompt-file updates across the `brainstorming`, `writing-plans`, and `subagent-driven-development` skills. Task-level ITC, task iteration, code-review, and spec-review loops are untouched.

**Tech Stack:** Markdown skill files, YAML schemas, no code. Verification is done via `Grep` checks that expected markers are present in the updated files.

**Spec reference:** `docs/superpowers/specs/2026-04-20-e2e-coverage-matrix-design.md`

---

## Task sequencing

Tasks 1–6 are independent text edits and can be done in any order. Task 7 (subagent-driven-development SKILL.md) depends on Tasks 4–6 so that its vocabulary matches the updated prompts. Recommended order: 1 → 2 → 3 → 4 → 5 → 6 → 7.

---

### Task 1: Create the testing-strategies playbook

**Files:**
- Create: `skills/subagent-driven-development/testing-strategies.md`

- [ ] **Step 1: Write the playbook file**

Create `skills/subagent-driven-development/testing-strategies.md` with exactly this content:

````markdown
# Testing Strategies Playbook

Canonical strategies the coverage matrix must address in every solution-level ITC. Column names in the matrix MUST match the section IDs below verbatim — do not invent new IDs inline; new strategies are introduced by appending sections to this file.

The testing-agent and coding-agent both walk this list for every journey listed in `<topic>-journeys.yaml` at solution-ITC negotiation time. Each journey × strategy cell is either a runnable scenario (with `command` and `assertion_shape`) or an explicit `na` with justification.

This list grows. New strategies are appended as new `##` sections. The coverage matrix requires a cell for every strategy currently in this file.

## happy_path

**When it applies:** Every journey. This is the baseline — the journey runs end-to-end with valid inputs from documented preconditions to the documented `expected_outcome`.

**Legitimate N/A reasons:** None. Every journey must have a happy_path scenario.

**Example scenario shape:**
- command: `npx playwright test tests/e2e/<journey>.spec.ts --grep @happy`
- assertion_shape: "Journey completes all steps; final DB state matches expected_outcome; UI shows success state; side-effects (emails, notifications) recorded."

**Common shallow mistakes:** Asserting only HTTP 200 or only element visible; skipping the DB-state check; not verifying side effects like emails actually sent.

## negative_path

**When it applies:** Any journey where an input is validated, an external service can fail, or a timeout is possible. In practice: nearly every journey.

**Legitimate N/A reasons:**
- Journey has no inputs and no external dependencies (rare; must cite specific steps).
- Negative-path coverage is fully subsumed by happy_path's assertions (e.g., an idempotent public read-only journey with no inputs).

**Example scenario shape:**
- command: `npx playwright test tests/e2e/<journey>.spec.ts --grep @negative`
- assertion_shape: "Invalid input at step X yields documented error response; no partial DB writes; no stale client state; user sees clear recovery path."

**Common shallow mistakes:** Only testing one negative case (e.g., just "invalid email"); asserting only status code without checking partial-write protection; missing service-failure simulation.

## state_persistence

**When it applies:** Journeys with 2+ steps, or any journey where intermediate state must survive navigation, reload, or session change.

**Legitimate N/A reasons:**
- Journey is a single atomic request with no client-side state to persist.

**Example scenario shape:**
- command: `npx playwright test tests/e2e/<journey>.spec.ts --grep @persistence`
- assertion_shape: "After reload mid-journey at step N, journey resumes from a valid state; authenticated session intact; partial data preserved or cleanly cleared per contract."

**Common shallow mistakes:** Only testing reload at the final step; not checking logout-login continuity; not asserting the specific state preserved.

## feature_interaction

**When it applies:** Any journey whose `feature_interactions` tag overlaps with another journey's tags.

**Legitimate N/A reasons:**
- Journey shares no `feature_interactions` tags with any other journey (truly isolated feature).

**Example scenario shape:**
- command: `npx playwright test tests/e2e/<journey>.spec.ts --grep @after-<other-journey-id>`
- assertion_shape: "Running journey J_other immediately before/after this journey: both end states correct; no shared state corruption; tag-overlapping features coexist."

**Common shallow mistakes:** Choosing an interacting journey at random rather than by shared `feature_interactions` tag; not running both journeys to completion; not asserting both end states.

## auth_boundary

**When it applies:** Any journey with authenticated steps, protected routes, or user-scoped data.

**Legitimate N/A reasons:**
- Journey is fully public; no authentication, no user-scoped data, no tenancy.

**Example scenario shape:**
- command: `npx playwright test tests/e2e/<journey>.spec.ts --grep @unauthorized`
- assertion_shape: "Unauthenticated actor hitting protected steps receives 401 or 403 (not 200); wrong-tenant actor cannot read or mutate data belonging to another tenant; no sensitive data in error response."

**Common shallow mistakes:** Testing only 'no token' but not 'wrong tenant'; asserting only response code without verifying no data leakage; skipping mutation endpoints and only testing reads.
````

- [ ] **Step 2: Verify the file exists with all 5 strategy sections**

Run (via Grep tool, output_mode: content):
- Pattern: `^## (happy_path|negative_path|state_persistence|feature_interaction|auth_boundary)$`
- Path: `skills/subagent-driven-development/testing-strategies.md`

Expected: 5 matches, one per strategy ID.

- [ ] **Step 3: Verify each strategy section has the 4 required subsections**

Run (via Grep tool, output_mode: count):
- Pattern: `\*\*(When it applies|Legitimate N/A reasons|Example scenario shape|Common shallow mistakes):\*\*`
- Path: `skills/subagent-driven-development/testing-strategies.md`

Expected: count = 20 (5 strategies × 4 subsections).

- [ ] **Step 4: Commit**

```bash
git add skills/subagent-driven-development/testing-strategies.md
git commit -m "Add testing-strategies playbook for coverage matrix

Five-strategy MVP (happy_path, negative_path, state_persistence,
feature_interaction, auth_boundary). Referenced by coding-agent and
testing-agent during solution-ITC negotiation; each journey × strategy
cell must resolve to a runnable scenario or justified N/A."
```

---

### Task 2: Update brainstorming skill — add Journey Enumeration step

**Files:**
- Modify: `skills/brainstorming/SKILL.md`

- [ ] **Step 1: Update the checklist in the `## Checklist` section**

Use Edit tool:

Old string:
```
You MUST create a task for each of these items and complete them in order:

1. **Explore project context** — check files, docs, recent commits
2. **Offer visual companion** (if topic will involve visual questions) — this is its own message, not combined with a clarifying question. See the Visual Companion section below.
3. **Ask clarifying questions** — one at a time, understand purpose/constraints/success criteria
4. **Propose 2-3 approaches** — with trade-offs and your recommendation
5. **Present design** — in sections scaled to their complexity, get user approval after each section
6. **Write design doc** — save to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` and commit
7. **Spec self-review** — quick inline check for placeholders, contradictions, ambiguity, scope (see below)
8. **User reviews written spec** — ask user to review the spec file before proceeding
9. **Transition to implementation** — invoke writing-plans skill to create implementation plan
```

New string:
```
You MUST create a task for each of these items and complete them in order:

1. **Explore project context** — check files, docs, recent commits
2. **Offer visual companion** (if topic will involve visual questions) — this is its own message, not combined with a clarifying question. See the Visual Companion section below.
3. **Ask clarifying questions** — one at a time, understand purpose/constraints/success criteria
4. **Propose 2-3 approaches** — with trade-offs and your recommendation
5. **Present design** — in sections scaled to their complexity, get user approval after each section
6. **Journey enumeration** — produce a draft `<topic>-journeys.yaml` (schema in `docs/superpowers/specs/2026-04-20-e2e-coverage-matrix-design.md` §4), present journeys with priorities to the user, and obtain approval. See the Journey Enumeration section below.
7. **Write design doc and `<topic>-journeys.yaml`** — save both to `docs/superpowers/specs/YYYY-MM-DD-<topic>-*` and commit in a single commit
8. **Spec self-review** — quick inline check for placeholders, contradictions, ambiguity, scope, and journey-completeness (see below)
9. **User reviews written spec** — ask user to review the spec file and journeys.yaml before proceeding
10. **Transition to implementation** — invoke writing-plans skill to create implementation plan
```

- [ ] **Step 2: Update the Process Flow graph to include Journey Enumeration**

Use Edit tool:

Old string:
```
    "Present design sections" -> "User approves design?";
    "User approves design?" -> "Present design sections" [label="no, revise"];
    "User approves design?" -> "Write design doc" [label="yes"];
    "Write design doc" -> "Spec self-review\n(fix inline)";
```

New string:
```
    "Present design sections" -> "User approves design?";
    "User approves design?" -> "Present design sections" [label="no, revise"];
    "User approves design?" -> "Journey enumeration" [label="yes"];
    "Journey enumeration" [shape=box];
    "User approves journeys?" [shape=diamond];
    "Journey enumeration" -> "User approves journeys?";
    "User approves journeys?" -> "Journey enumeration" [label="no, revise"];
    "User approves journeys?" -> "Write design doc" [label="yes"];
    "Write design doc" -> "Spec self-review\n(fix inline)";
```

- [ ] **Step 3: Add Journey Enumeration process section**

Use Edit tool:

Old string:
```
**Working in existing codebases:**

- Explore the current structure before proposing changes. Follow existing patterns.
- Where existing code has problems that affect the work (e.g., a file that's grown too large, unclear boundaries, tangled responsibilities), include targeted improvements as part of the design - the way a good developer improves code they're working in.
- Don't propose unrelated refactoring. Stay focused on what serves the current goal.

## After the Design
```

New string:
```
**Working in existing codebases:**

- Explore the current structure before proposing changes. Follow existing patterns.
- Where existing code has problems that affect the work (e.g., a file that's grown too large, unclear boundaries, tangled responsibilities), include targeted improvements as part of the design - the way a good developer improves code they're working in.
- Don't propose unrelated refactoring. Stay focused on what serves the current goal.

## Journey Enumeration

After the design is approved and before writing the spec, walk the PRD and the design's feature list to enumerate the end-to-end flows a user takes across the features. Each journey is a first-class artifact — it will drive the solution-level coverage matrix at implementation time (see `docs/superpowers/specs/2026-04-20-e2e-coverage-matrix-design.md`).

**Produce** a `<topic>-journeys.yaml` draft using this schema:

```yaml
journeys:
  - id: J1
    name: "<short human-readable flow name>"
    persona: "<who is taking the journey and any relevant attribute>"
    preconditions: "<system state before the journey begins>"
    steps:
      - "<ordered action>"
      - "<ordered action>"
    expected_outcome: "<concrete end state — DB, UI, side effects>"
    feature_interactions: [<tags drawn from the design's feature list>]
    priority: P0   # P0 must-cover | P1 should-cover | P2 nice-to-have
```

**Present** the journeys to the user with priorities and obtain approval using the same gate pattern as the rest of the design.

**Rules**
- `feature_interactions` tags MUST come from the design's declared feature list; do not invent tags.
- Each P0 journey MUST have specific steps and a concrete `expected_outcome`.
- No journey should be so vague that a strategy cell (see testing-strategies playbook) would be unwriteable.

## After the Design
```

- [ ] **Step 4: Update Documentation section to write journeys.yaml**

Use Edit tool:

Old string:
```
**Documentation:**

- Write the validated design (spec) to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`
  - (User preferences for spec location override this default)
- Use elements-of-style:writing-clearly-and-concisely skill if available
- Commit the design document to git
```

New string:
```
**Documentation:**

- Write the validated design (spec) to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`
- Write the approved journeys to `docs/superpowers/specs/YYYY-MM-DD-<topic>-journeys.yaml`
  - (User preferences for spec location override these defaults)
- Use elements-of-style:writing-clearly-and-concisely skill if available
- Commit both files to git in a single commit
```

- [ ] **Step 5: Extend Spec Self-Review with journey checks**

Use Edit tool:

Old string:
```
**Spec Self-Review:**
After writing the spec document, look at it with fresh eyes:

1. **Placeholder scan:** Any "TBD", "TODO", incomplete sections, or vague requirements? Fix them.
2. **Internal consistency:** Do any sections contradict each other? Does the architecture match the feature descriptions?
3. **Scope check:** Is this focused enough for a single implementation plan, or does it need decomposition?
4. **Ambiguity check:** Could any requirement be interpreted two different ways? If so, pick one and make it explicit.

Fix any issues inline. No need to re-review — just fix and move on.
```

New string:
```
**Spec Self-Review:**
After writing the spec document and journeys.yaml, look at both with fresh eyes:

1. **Placeholder scan:** Any "TBD", "TODO", incomplete sections, or vague requirements? Fix them.
2. **Internal consistency:** Do any sections contradict each other? Does the architecture match the feature descriptions?
3. **Scope check:** Is this focused enough for a single implementation plan, or does it need decomposition?
4. **Ambiguity check:** Could any requirement be interpreted two different ways? If so, pick one and make it explicit.
5. **Journey completeness:** Every P0 journey has non-trivial steps and a specific `expected_outcome`. `feature_interactions` tags are drawn from the design's declared feature list (no invented tags). No journey is so vague that a strategy cell would be unwriteable.

Fix any issues inline. No need to re-review — just fix and move on.
```

- [ ] **Step 6: Update User Review Gate to mention journeys.yaml**

Use Edit tool:

Old string:
```
> "Spec written and committed to `<path>`. Please review it and let me know if you want to make any changes before we start writing out the implementation plan."
```

New string:
```
> "Spec written and committed to `<spec-path>` and `<journeys-path>`. Please review both and let me know if you want to make any changes before we start writing out the implementation plan."
```

- [ ] **Step 7: Verify the edits landed**

Run (via Grep tool, output_mode: content):
- Pattern: `Journey enumeration|<topic>-journeys\.yaml|Journey Enumeration`
- Path: `skills/brainstorming/SKILL.md`

Expected: matches for each pattern — at least the new checklist item, the new Process Flow node, the new process section heading, the new Documentation bullet.

- [ ] **Step 8: Commit**

```bash
git add skills/brainstorming/SKILL.md
git commit -m "Add Journey Enumeration step to brainstorming skill

Brainstorm now produces journeys.yaml alongside the design doc;
spec-review includes journey-completeness checks; user-review gate
covers both artifacts."
```

---

### Task 3: Update writing-plans skill — plan schema additions

**Files:**
- Modify: `skills/writing-plans/SKILL.md`

- [ ] **Step 1: Extend the Plan Document Header template to include Journeys reference**

Use Edit tool:

Old string:
````
**Every plan MUST start with this header:**

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```
````

New string:
````
**Every plan MUST start with this header:**

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

**If the spec has a companion `<topic>-journeys.yaml`, include a `## Journeys (reference)` section immediately after the header:**

```markdown
## Journeys (reference)

- J1 — <name from journeys.yaml>
- J2 — <name from journeys.yaml>
- J3 — <name from journeys.yaml>

(Full detail lives in `docs/superpowers/specs/<date>-<topic>-journeys.yaml`.)
```

This is a compact readability hint for plan readers — do not duplicate the full journey content.
````

- [ ] **Step 2: Add `contributes_to` field to the Task Structure template**

Use Edit tool:

Old string:
````
## Task Structure

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`
````

New string:
````
## Task Structure

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

**Contributes to:** [J1, J3]   <!-- optional; include only if this task clearly implements one or more journeys from journeys.yaml. Omit if unclear. Not a gate. -->
````

- [ ] **Step 3: Verify the edits landed**

Run (via Grep tool, output_mode: content):
- Pattern: `Journeys \(reference\)|Contributes to:|contributes_to`
- Path: `skills/writing-plans/SKILL.md`

Expected: matches for `Journeys (reference)` in the plan header section and `Contributes to:` in the task structure.

- [ ] **Step 4: Commit**

```bash
git add skills/writing-plans/SKILL.md
git commit -m "Add Journeys reference section and contributes_to field

Plans now include a compact journey index in the header and optionally
cite per-task journey contributions. Source of truth remains
journeys.yaml; these fields are readability/traceability hints."
```

---

### Task 4: Rewrite coding-agent solution-ITC block — coverage matrix

**Files:**
- Modify: `skills/subagent-driven-development/coding-agent-prompt.md`

- [ ] **Step 1: Replace the "If Producing a Solution ITC" block**

Use Edit tool:

Old string:
````
    ## If Producing a Solution ITC

    When dispatched for solution-level ITC negotiation (after all tasks complete),
    scan the actual implementation first, then use this YAML format instead:

    ```yaml
    solution: "[feature or project name]"
    tasks_covered: [1, 2, 3]  # list all completed task IDs

    implementation_summary:
      entry_points:
        - "[HTTP method] [route or component path]"
      protected_routes:  # omit if not applicable
        - "[HTTP method] [route]"

    test_contract:
      tiers_required: [e2e, full_suite]
      rationale: "[one sentence: all tasks complete, verify full user journey end-to-end]"

      e2e:
        commands:
          - "[exact E2E command — e.g., npx playwright test tests/e2e/feature.spec.ts]"
        scenarios:
          - "[user flow to verify — e.g., 'user login and session persistence']"
        required_services:
          - "[what must be running — e.g., 'backend API on port 3001', 'database seeded', 'frontend built']"

      full_suite:
        command: "[exact command to run full test suite — e.g., npm test]"
        rationale: "Catch any cross-task regression not visible in individual task ITCs"

    acceptance_criteria:
      e2e: all scenarios pass
      full_suite: all pass
      no_regression: true

    sign_off:
      coding_agent: ✅
      testing_agent: (pending)
    ```
````

New string:
````
    ## If Producing a Solution ITC

    When dispatched for solution-level ITC negotiation (after all tasks complete):

    **Required inputs** (all three must be loaded and referenced before drafting):
    1. The project's `<topic>-journeys.yaml` (the dispatcher pastes it in full below).
    2. The playbook at `skills/subagent-driven-development/testing-strategies.md` (the dispatcher pastes it in full below).
    3. The actual implementation — scan the codebase for entry points, protected routes, and test files.

    Your output is a **coverage matrix**: rows are journey IDs from journeys.yaml, columns are strategy IDs from the playbook. Every cell MUST be either a `scenario` (with `command` and `assertion_shape`) or an `na` with a non-empty justification string. Do NOT invent strategy IDs — use the canonical IDs from the playbook verbatim.

    Use this YAML format:

    ```yaml
    solution: "[feature or project name]"
    tasks_covered: [1, 2, 3]  # list all completed task IDs

    implementation_summary:
      entry_points:
        - "[HTTP method] [route or component path]"
      protected_routes:  # omit if not applicable
        - "[HTTP method] [route]"

    test_contract:
      tiers_required: [e2e, full_suite]
      rationale: "[one sentence: all tasks complete, verify full user journey end-to-end via coverage matrix]"

      e2e:
        required_services:
          - "[what must be running — e.g., 'backend API on port 3001', 'database seeded', 'frontend built']"

        coverage_matrix:
          J1:
            happy_path:
              scenario:
                command: "[exact runnable command, e.g., npx playwright test tests/e2e/<file>.spec.ts --grep @J1-happy]"
                assertion_shape: "[concrete deep property — e.g., 'final DB state: both users in workspace, roles=[owner,member], invite marked accepted']"
            negative_path:
              scenario:
                command: "..."
                assertion_shape: "..."
            state_persistence:
              scenario:
                command: "..."
                assertion_shape: "..."
            feature_interaction:
              scenario:
                command: "..."
                assertion_shape: "..."
            auth_boundary:
              scenario:
                command: "..."
                assertion_shape: "..."
          J2:
            happy_path:    { scenario: { command: "...", assertion_shape: "..." } }
            negative_path: { scenario: { command: "...", assertion_shape: "..." } }
            state_persistence: { na: "Journey is a single atomic request with no multi-step state to persist." }
            feature_interaction: { scenario: { command: "...", assertion_shape: "..." } }
            auth_boundary: { na: "Journey is fully public; no authentication or user-scoped data." }

      full_suite:
        command: "[exact command to run full test suite — e.g., npm test]"
        rationale: "Catch any cross-task regression not visible in individual task ITCs"

    acceptance_criteria:
      e2e: every scenario cell PASS; every na cell has non-empty justification
      full_suite: all pass
      no_regression: true

    sign_off:
      coding_agent: ✅
      testing_agent: (pending)
    ```

    **Rules for the matrix**
    - Every journey in the pasted journeys.yaml MUST be a row.
    - Every strategy ID currently in the pasted testing-strategies.md MUST be a column for every row.
    - Every cell = `scenario` or `na`. No empty cells.
    - `assertion_shape` is required prose describing the deep property the scenario asserts; avoid shallow shapes like "returns 200" or "element visible" alone — they will be rejected.
    - `na` justification must cite a legitimate reason from the playbook (structurally inapplicable, covered by another cell, or explicitly deferred). Lazy `na` ("too hard", "not critical", one-word skip) will be rejected.

    ## Journeys.yaml (paste in full)

    [Dispatcher: paste the full contents of `docs/superpowers/specs/<date>-<topic>-journeys.yaml` here]

    ## Testing Strategies Playbook (paste in full)

    [Dispatcher: paste the full contents of `skills/subagent-driven-development/testing-strategies.md` here]
````

- [ ] **Step 2: Verify the edits landed**

Run (via Grep tool, output_mode: content):
- Pattern: `coverage_matrix|testing-strategies\.md|Journeys\.yaml \(paste in full\)`
- Path: `skills/subagent-driven-development/coding-agent-prompt.md`

Expected: matches for all three patterns.

- [ ] **Step 3: Verify the old scenarios: field shape is gone from solution-ITC block**

Run (via Grep tool, output_mode: content):
- Pattern: `scenarios:\s*\n\s*-`
- Path: `skills/subagent-driven-development/coding-agent-prompt.md`
- Multiline: true

Expected: no matches (the old free-form scenarios list has been replaced by coverage_matrix).

- [ ] **Step 4: Commit**

```bash
git add skills/subagent-driven-development/coding-agent-prompt.md
git commit -m "Rewrite coding-agent solution-ITC block for coverage matrix

Coding-agent now loads journeys.yaml and testing-strategies.md
before drafting the solution ITC and produces a Journey × Strategy
coverage_matrix instead of a free-form scenarios list. Every cell
must resolve to a runnable scenario or justified N/A."
```

---

### Task 5: Rewrite testing-agent solution-ITC review block — matrix review rubric

**Files:**
- Modify: `skills/subagent-driven-development/testing-agent-prompt.md`

- [ ] **Step 1: Replace the "If Reviewing a Solution ITC" block**

Use Edit tool:

Old string:
````
    ## If Reviewing a Solution ITC

    When the coding agent submits a solution ITC (contains `solution:` field, `tiers_required: [e2e, full_suite]`), check these instead of the task ITC checks above:

    1. **implementation_summary is accurate**
       - Do the listed entry_points and routes match what was actually built?
       - Scan the codebase to verify — do not trust the coding agent's self-report.

    2. **E2E scenarios cover all user-facing flows**
       - Is there a scenario for every significant user journey?
       - Are error flows covered (not just happy path)?
       - Does every scenario have an exact, runnable command?

    3. **required_services is complete**
       - What must be running for E2E tests to execute?
       - Are all services, databases, and build steps listed?

    4. **full_suite command is correct**
       - Does the command run the complete test suite (not just a subset)?
       - Is it the canonical command (check package.json scripts or Makefile)?
````

New string:
````
    ## If Reviewing a Solution ITC

    When the coding agent submits a solution ITC (contains `solution:` field, `tiers_required: [e2e, full_suite]`), your job is to validate the **coverage matrix** against the project's journeys and the testing-strategies playbook.

    **Required reading before review** (the dispatcher pastes these in full below):
    1. `<topic>-journeys.yaml` — the authoritative journey list.
    2. `skills/subagent-driven-development/testing-strategies.md` — the canonical strategy playbook, especially the `Common shallow mistakes` and `Legitimate N/A reasons` per strategy. These sections are your review ammunition.

    Check all of the following:

    1. **implementation_summary is accurate**
       - Do listed entry_points and routes match what was actually built?
       - Scan the codebase to verify — do not trust the coding agent's self-report.

    2. **Matrix shape is complete**
       - Every journey in the pasted journeys.yaml is present as a row.
       - Every strategy ID currently in the pasted playbook is present as a column for every row.
       - Every cell is either `scenario` (with `command` and `assertion_shape`) or `na` (with non-empty justification). No empty cells.

    3. **No shallow `assertion_shape`** — reject any of the following patterns:
       - "returns 200" / "HTTP 200" alone
       - "element visible" alone
       - Status-code checks with no DB, state, or side-effect verification
       - Vague assertions like "works correctly" / "behaves as expected"
       For each such cell, provide the exact deeper assertion the scenario should make.

    4. **No lazy `na`** — reject any `na` justification that does not match a legitimate reason from the playbook:
       - Structurally inapplicable (e.g., `auth_boundary` on a fully public journey)
       - Covered by another cell (e.g., `happy_path` already asserts the property)
       - Explicitly deferred in the spec (must cite the spec section)
       Reject: "too hard", "not critical", "skip", or any one-word justification.

    5. **Commands are runnable and tied to real code paths**
       - Every `scenario.command` is copy-pasteable.
       - Every command targets a test file or `--grep` tag that actually exists (or is clearly named so the implementer will create it). Scan the codebase to confirm where necessary.

    6. **required_services is complete**
       - What must be running for the matrix to execute end-to-end?
       - Are all services, databases, seeded data, and build steps listed?

    7. **full_suite command is correct**
       - Does the command run the complete test suite (not just a subset)?
       - Is it the canonical command (check package.json scripts or Makefile)?

    ## Journeys.yaml (paste in full)

    [Dispatcher: paste the full contents of `docs/superpowers/specs/<date>-<topic>-journeys.yaml` here]

    ## Testing Strategies Playbook (paste in full)

    [Dispatcher: paste the full contents of `skills/subagent-driven-development/testing-strategies.md` here]
````

- [ ] **Step 2: Verify the edits landed**

Run (via Grep tool, output_mode: content):
- Pattern: `coverage matrix|No shallow|No lazy|testing-strategies\.md`
- Path: `skills/subagent-driven-development/testing-agent-prompt.md`

Expected: matches for all four patterns.

- [ ] **Step 3: Commit**

```bash
git add skills/subagent-driven-development/testing-agent-prompt.md
git commit -m "Rewrite testing-agent solution-ITC review for coverage matrix

Review now validates matrix shape completeness, rejects shallow
assertion_shapes (HTTP 200 alone, element-visible alone), rejects
lazy na justifications, and requires pasted journeys.yaml +
testing-strategies.md as inputs."
```

---

### Task 6: Update test-runner-solution prompt — matrix execution and report

**Files:**
- Modify: `skills/subagent-driven-development/test-runner-solution-prompt.md`

- [ ] **Step 1: Replace the body of the prompt to walk the matrix**

Use Edit tool:

Old string:
````
    ## Required Stack

    Verify each of the following is running before executing any command:
    [Paste solution_ITC.test_contract.e2e.required_services — one per line]

    If any service is not running or the frontend is not built:
    Report BLOCKED immediately with exactly what is missing. Do not run tests.

    ## E2E Commands

    [Paste solution_ITC.test_contract.e2e.commands — one per line]

    ## Full Suite Command

    [Paste solution_ITC.test_contract.full_suite.command]

    ## E2E Scenarios to Verify

    [Paste solution_ITC.test_contract.e2e.scenarios — one per line]

    ## Acceptance Criteria

    [Paste solution_ITC.acceptance_criteria]

    ## Playwright Options

    For E2E and full suite tests you have two execution modes — choose based on availability:

    **playwright-cli** (preferred when Playwright is installed in the project):
    - Run commands from the `## E2E Commands` and `## Full Suite Command` sections directly via Bash
    - Example: `npx playwright test tests/e2e/auth.spec.ts`
    - Use this when the ITC provides explicit test file commands

    **playwright-mcp** (use when playwright-cli is unavailable or commands fail to run):
    - Use the browser MCP tools (`browser_navigate`, `browser_click`, `browser_fill_form`, `browser_snapshot`, etc.) to manually walk through each scenario in `## E2E Scenarios to Verify`
    - Navigate to the app URL, interact with the UI step by step, and verify expected outcomes
    - Use `browser_snapshot` or `browser_take_screenshot` to capture evidence of pass/fail
    - Report each scenario individually based on what you observed

    If neither is available, report BLOCKED with reason "playwright-cli not installed and playwright-mcp not available".

    ## Your Job

    1. Verify full stack is running (check each required service).
       Report BLOCKED immediately if anything is missing — do not run tests.
    2. Choose execution mode (see Playwright Options above).
    3. Run E2E tests. For each scenario in the scenarios list, note pass or fail.
    4. Run full suite command (playwright-cli) or verify all scenarios (playwright-mcp).
    5. Report results in the format below. Do NOT fix failures.

    ## Report Format

    Status: PASS | FAIL | BLOCKED

    E2E Results:
      - scenario: "[scenario name from scenarios list]"
        status: PASS | FAIL
        failure_detail: |  # omit if PASS
          [What user action failed. What was expected vs actual.
           Which test file and line, if available.
           Enough detail for the implementer to know which flow broke
           and where to start debugging.]

    Full Suite:
      status: PASS | FAIL
      summary: "[X/Y tests passed]"
      failures:  # omit if PASS
        - file: "[test file path]"
          test: "[test name]"
          error: "[error message]"

    If BLOCKED — use this format instead:
    Status: BLOCKED
    reason: "[exactly what is not running or missing]"
    resolution: "[what must be done before re-running — e.g., 'start the API server on port 3001', 'run npm run build', 'seed the test database']"
````

New string:
````
    ## Required Stack

    Verify each of the following is running before executing any command:
    [Paste solution_ITC.test_contract.e2e.required_services — one per line]

    If any service is not running or the frontend is not built:
    Report BLOCKED immediately with exactly what is missing. Do not run tests.

    ## Coverage Matrix

    [Paste solution_ITC.test_contract.e2e.coverage_matrix verbatim — preserve journey IDs and strategy IDs exactly]

    ## Full Suite Command

    [Paste solution_ITC.test_contract.full_suite.command]

    ## Acceptance Criteria

    [Paste solution_ITC.acceptance_criteria]

    ## Playwright Options

    For matrix scenarios and the full suite command you have two execution modes — choose based on availability:

    **playwright-cli** (preferred when Playwright is installed in the project):
    - Run each cell's `scenario.command` directly via Bash (e.g., `npx playwright test tests/e2e/auth.spec.ts --grep @J1-happy`)
    - Run the full suite command directly via Bash

    **playwright-mcp** (use when playwright-cli is unavailable or commands fail to run):
    - Use the browser MCP tools (`browser_navigate`, `browser_click`, `browser_fill_form`, `browser_snapshot`, etc.) to walk through each cell's `assertion_shape` manually
    - For each `scenario` cell, navigate the app per the `assertion_shape` and verify the deep property stated there
    - Use `browser_snapshot` or `browser_take_screenshot` to capture evidence of pass/fail
    - Report each cell individually based on what you observed

    If neither is available, report BLOCKED with reason "playwright-cli not installed and playwright-mcp not available".

    ## Your Job

    1. Verify full stack is running (check each required service).
       Report BLOCKED immediately if anything is missing — do not run tests.
    2. Choose execution mode (see Playwright Options above).
    3. Walk the coverage matrix row-by-row, column-by-column (journey-major). For each cell:
       - If `scenario`: run the command, capture stdout/stderr, classify as PASS or FAIL. On FAIL, record the `assertion_shape` for implementer feedback.
       - If `na`: do NOT execute. Echo the justification text into the report.
    4. Run the full suite command once.
    5. Report results in the format below. Do NOT fix failures.

    ## Report Format

    Status: PASS | FAIL | BLOCKED

    matrix_results:
      J1:
        happy_path: { status: PASS }
        negative_path:
          status: FAIL
          assertion_shape: "[echoed from input matrix cell]"
          failure_detail: |
            [What failed. What was expected vs actual. Test file and line.
             Enough detail for the implementer to know which property broke.]
        state_persistence: { status: PASS }
        feature_interaction: { status: PASS }
        auth_boundary:
          status: NA
          justification: "[echoed verbatim from input matrix cell]"
      J2: ...

    full_suite:
      status: PASS | FAIL
      summary: "[X/Y tests passed]"
      failures:  # omit if PASS
        - file: "[test file path]"
          test: "[test name]"
          error: "[error message]"

    overall: PASS | FAIL

    **Overall verdict rule:**
    - PASS iff every `scenario` cell is PASS, every `na` cell has a non-empty justification echoed, and full_suite is PASS.
    - FAIL iff any `scenario` cell is FAIL, or any `na` cell has empty justification, or full_suite is FAIL.

    If BLOCKED — use this format instead:
    Status: BLOCKED
    reason: "[exactly what is not running or missing]"
    resolution: "[what must be done before re-running — e.g., 'start the API server on port 3001', 'run npm run build', 'seed the test database']"
````

- [ ] **Step 2: Verify the edits landed**

Run (via Grep tool, output_mode: content):
- Pattern: `Coverage Matrix|matrix_results|overall: PASS`
- Path: `skills/subagent-driven-development/test-runner-solution-prompt.md`

Expected: matches for all three patterns.

- [ ] **Step 3: Verify old scenario-list structure is gone**

Run (via Grep tool, output_mode: content):
- Pattern: `E2E Scenarios to Verify|E2E Commands|E2E Results:`
- Path: `skills/subagent-driven-development/test-runner-solution-prompt.md`

Expected: no matches (old section headers have been replaced).

- [ ] **Step 4: Commit**

```bash
git add skills/subagent-driven-development/test-runner-solution-prompt.md
git commit -m "Update solution test-runner to walk coverage matrix

Runner now consumes the coverage matrix, executes each scenario cell,
echoes na justifications, and emits matrix_results mirroring input
shape. Adds explicit overall: PASS|FAIL verdict rule tied to per-cell
outcomes."
```

---

### Task 7: Update subagent-driven-development SKILL — glue changes

**Files:**
- Modify: `skills/subagent-driven-development/SKILL.md`

- [ ] **Step 1: Extend the Solution ITC section with required inputs and post-negotiation gate**

Use Edit tool:

Old string:
```
### Solution ITC

Negotiated once after all tasks complete, before the E2E harness runs. The coding agent scans the **actual implementation** (not the plan) to produce accurate entry points. The testing agent proposes E2E scenarios and full-suite commands based on what was actually built. The same five-round protocol applies: negotiation alternates between coding-agent (odd rounds) and testing-agent (even rounds) up to Round 5. The flowchart's escalation arc represents only the terminal case (5 rounds without agreement).

See full contract structure: `docs/superpowers/specs/2026-04-14-e2e-test-harness-design.md`
```

New string:
```
### Solution ITC

Negotiated once after all tasks complete, before the E2E harness runs.

**Required inputs** (load all three before dispatching Round 1):
1. `docs/superpowers/specs/<date>-<topic>-journeys.yaml` — authoritative journey list, produced at brainstorm time.
2. `skills/subagent-driven-development/testing-strategies.md` — the canonical strategy playbook.
3. The actual implementation (coding agent scans the codebase, not the plan).

**Output shape:** instead of a free-form `scenarios:` list, the signed solution ITC contains a `coverage_matrix` whose rows are journey IDs and whose columns are the canonical strategy IDs from the playbook. Every cell is either a `scenario` (with `command` and `assertion_shape`) or an `na` (with non-empty justification).

**Protocol:** same five-round negotiation alternating between coding-agent (odd rounds) and testing-agent (even rounds) up to Round 5. The flowchart's escalation arc represents the terminal case (5 rounds without agreement).

**Post-negotiation gate (before dispatching the solution test-runner):**
Verify the signed ITC's `coverage_matrix` has:
- A row for every journey in journeys.yaml.
- A column for every strategy ID currently in testing-strategies.md.
- A non-empty `scenario` or `na` value in every cell.

If any row, column, or cell is missing, reject the negotiation as incomplete and re-dispatch Round 1.

See full contract structure: `docs/superpowers/specs/2026-04-20-e2e-coverage-matrix-design.md`
```

- [ ] **Step 2: Update the example workflow to reflect matrix output**

Use Edit tool:

Old string:
```
[After all tasks complete]

[Dispatch coding-agent Solution ITC Round 1 — scans actual implementation: entry points, protected routes]
Coding agent: Documents real routes built. Proposes e2e + full_suite tiers.
              coding_agent: ✅

[Dispatch testing-agent Solution ITC Round 2 — task spec + coding agent's draft]
Testing agent: ✅ Approved — E2E scenarios cover all user flows, full suite command confirmed.

[Write docs/superpowers/contracts/2026-04-14T16-00-00-solution_itc.md and commit]

[Dispatch E2E + full suite test-runner — commands from solution ITC]
Test runner:
  Status: PASS
  E2E Results:
    - scenario: "user registration + email verification"
      status: PASS
    - scenario: "login and session persistence"
      status: PASS
    - scenario: "protected route access with valid token"
      status: PASS
  Full Suite:
    status: PASS
    summary: "47/47 tests passed"
```

New string:
```
[After all tasks complete]

[Load docs/superpowers/specs/<date>-<topic>-journeys.yaml and skills/subagent-driven-development/testing-strategies.md]

[Dispatch coding-agent Solution ITC Round 1 — scans actual implementation + journeys.yaml + playbook]
Coding agent: Documents real routes built. Produces coverage_matrix with a row per journey
              and a column per strategy ID (happy_path, negative_path, state_persistence,
              feature_interaction, auth_boundary). Every cell = scenario or justified na.
              coding_agent: ✅

[Dispatch testing-agent Solution ITC Round 2 — task spec + draft matrix + journeys.yaml + playbook]
Testing agent: ✅ Approved — matrix complete, no shallow assertion_shapes, na justifications
               all structurally valid per playbook.

[Post-negotiation gate: verify matrix has row per journey × column per strategy, no empty cells]

[Write docs/superpowers/contracts/2026-04-20T16-00-00-solution_itc.md and commit]

[Dispatch E2E + full suite test-runner — coverage_matrix + full_suite command]
Test runner:
  Status: PASS
  matrix_results:
    J1:
      happy_path: { status: PASS }
      negative_path: { status: PASS }
      state_persistence: { status: PASS }
      feature_interaction: { status: PASS }
      auth_boundary: { status: PASS }
    J2:
      happy_path: { status: PASS }
      negative_path: { status: PASS }
      state_persistence: { status: NA, justification: "Journey is a single atomic request." }
      feature_interaction: { status: PASS }
      auth_boundary: { status: NA, justification: "Fully public journey, no auth surface." }
  full_suite:
    status: PASS
    summary: "47/47 tests passed"
  overall: PASS
```

- [ ] **Step 3: Verify the edits landed**

Run (via Grep tool, output_mode: content):
- Pattern: `coverage_matrix|Post-negotiation gate|testing-strategies\.md|matrix_results`
- Path: `skills/subagent-driven-development/SKILL.md`

Expected: matches for all four patterns.

- [ ] **Step 4: Commit**

```bash
git add skills/subagent-driven-development/SKILL.md
git commit -m "Wire coverage matrix into subagent-driven-development skill

Solution ITC now lists journeys.yaml and testing-strategies.md as
required inputs, documents the coverage_matrix output shape, adds a
post-negotiation completeness gate, and updates the example workflow
to show matrix_results output."
```

---

## Self-review (post-plan checklist)

After the plan is written, the author (this plan's writer) runs this checklist — no subagent dispatch:

1. **Spec coverage:** Walk every section of the design spec and confirm a task implements it.
   - Section 3 (architecture): ✅ Tasks 1–7 cover all additions and modifications.
   - Section 4 (journeys.yaml schema): ✅ Task 2 writes the schema into brainstorming skill.
   - Section 5 (strategy playbook): ✅ Task 1.
   - Section 6 (coverage matrix shape): ✅ Tasks 4, 5, 6 all reference the Section 6 schema.
   - Section 7.1 (brainstorming): ✅ Task 2.
   - Section 7.2 (writing-plans): ✅ Task 3.
   - Section 7.3 (subagent-driven-development SKILL): ✅ Task 7.
   - Section 7.4 (coding-agent): ✅ Task 4.
   - Section 7.5 (testing-agent): ✅ Task 5.
   - Section 7.6 (test-runner-solution): ✅ Task 6.
   - Section 8 (feedback loop): Unchanged by plan — `assertion_shape` is echoed in the Task 6 report format, which realizes the feedback-loop mechanism.
   - Section 9 (file list): ✅ All added / modified files covered; unchanged files are not touched.
   - Section 10 (risks): No task required — risks are documentation.
   - Section 11 (deferred): No task required — deferred items are out of scope.

2. **Placeholder scan:** No "TBD", "TODO", "implement later" in the plan. Each edit shows the exact `old_string` and `new_string` content.

3. **Type consistency:** Strategy IDs used in every task (`happy_path`, `negative_path`, `state_persistence`, `feature_interaction`, `auth_boundary`) match exactly. Field names (`scenario`, `na`, `command`, `assertion_shape`, `justification`, `coverage_matrix`, `matrix_results`, `overall`) match exactly across tasks 1, 4, 5, 6, and 7.

4. **Commit-per-task:** Every task ends with a commit step with a specific message.
