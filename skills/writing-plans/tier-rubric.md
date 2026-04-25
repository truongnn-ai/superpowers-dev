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
