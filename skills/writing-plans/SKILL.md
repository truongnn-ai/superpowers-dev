---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
---

# Writing Plans

## Overview

Write comprehensive implementation plans assuming the engineer has zero context for our codebase and questionable taste. Document everything they need to know: which files to touch for each task, code, testing, docs they might need to check, how to test it. Give them the whole plan as bite-sized tasks. DRY. YAGNI. TDD. Frequent commits.

Assume they are a skilled developer, but know almost nothing about our toolset or problem domain. Assume they don't know good test design very well.

**Announce at start:** "I'm using the writing-plans skill to create the implementation plan."

**Context:** This should be run in a dedicated worktree (created by brainstorming skill).

**Save plans to:** `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- (User preferences for plan location override this default)

## Scope Check

If the spec covers multiple independent subsystems, it should have been broken into sub-project specs during brainstorming. If it wasn't, suggest breaking this into separate plans — one per subsystem. Each plan should produce working, testable software on its own.

## File Structure

Before defining tasks, map out which files will be created or modified and what each one is responsible for. This is where decomposition decisions get locked in.

- Design units with clear boundaries and well-defined interfaces. Each file should have one clear responsibility.
- You reason best about code you can hold in context at once, and your edits are more reliable when files are focused. Prefer smaller, focused files over large ones that do too much.
- Files that change together should live together. Split by responsibility, not by technical layer.
- In existing codebases, follow established patterns. If the codebase uses large files, don't unilaterally restructure - but if a file you're modifying has grown unwieldy, including a split in the plan is reasonable.

This structure informs the task decomposition. Each task should produce self-contained changes that make sense independently.

## Bite-Sized Task Granularity

**Each step is one action (2-5 minutes):**
- "Write the failing test" - step
- "Run it to make sure it fails" - step
- "Implement the minimal code to make the test pass" - step
- "Run the tests and make sure they pass" - step
- "Commit" - step

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

## Plan Document Header

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

(Full detail lives in `docs/superpowers/specs/<date>-<topic>-journeys.yaml`.)
```

This is a compact readability hint for plan readers — do not duplicate full journey content here.

## Task Structure

````markdown
### Task N: [Component Name]

```yaml
tier: heavy
tier_reason: "H1: adds new behavior — implements feature X end-to-end"
```

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

**Contributes to:** [J1, J3]   <!-- optional: include only when this task clearly implements journeys from journeys.yaml; omit if unclear -->

- [ ] **Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

- [ ] **Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## No Placeholders

Every step must contain the actual content an engineer needs. These are **plan failures** — never write them:
- "TBD", "TODO", "implement later", "fill in details"
- "Add appropriate error handling" / "add validation" / "handle edge cases"
- "Write tests for the above" (without actual test code)
- "Similar to Task N" (repeat the code — the engineer may be reading tasks out of order)
- Steps that describe what to do without showing how (code blocks required for code steps)
- References to types, functions, or methods not defined in any task

## Remember
- Exact file paths always
- Complete code in every step — if a step changes code, show the code
- Exact commands with expected output
- DRY, YAGNI, TDD, frequent commits

## Self-Review

After writing the complete plan, look at the spec with fresh eyes and check the plan against it. This is a checklist you run yourself — not a subagent dispatch.

**1. Spec coverage:** Skim each section/requirement in the spec. Can you point to a task that implements it? List any gaps.

**2. Placeholder scan:** Search your plan for red flags — any of the patterns from the "No Placeholders" section above. Fix them.

**3. Type consistency:** Do the types, method signatures, and property names you used in later tasks match what you defined in earlier tasks? A function called `clearLayers()` in Task 3 but `clearFullLayers()` in Task 7 is a bug.

**4. Tier coverage:** Every task has a `tier` block with `tier` and `tier_reason`. Every `tier_reason` quotes a clause id from `./tier-rubric.md` (e.g., `T1`, `S2`, `H4`). Tasks with no clear match are tagged `heavy` citing `H5`.

If you find issues, fix them inline. No need to re-review — just fix and move on. If you find a spec requirement with no task, add the task.

## Execution Handoff

After saving the plan, offer execution choice:

**"Plan complete and saved to `docs/superpowers/plans/<filename>.md`. Two execution options:**

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

**Which approach?"**

**If Subagent-Driven chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:subagent-driven-development
- Fresh subagent per task + two-stage review

**If Inline Execution chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:executing-plans
- Batch execution with checkpoints for review
