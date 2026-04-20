# Daily Workflows

The 4 flows you'll use every day. Each one starts with you describing the task — the agent handles the rest.

---

## 1. Building a new feature

**Start with:** "I want to build / add / implement X"

```
brainstorming       ← agent asks questions, you answer
    ↓
writing-plans       ← agent writes step-by-step plan, you approve
    ↓
using-git-worktrees ← agent creates an isolated branch
    ↓
subagent-driven-development  ← agent executes each task with TDD + code review
    ↓
finishing-a-development-branch ← agent asks: merge, PR, or keep for now?
```

**Your checkpoints:** Approve the design. Approve the plan. Choose how to finish.

---

## 2. Fixing a bug

**Start with:** "There's a bug where X happens when Y" or "This test is failing"

```
systematic-debugging        ← agent investigates root cause before touching code
    ↓
test-driven-development     ← agent writes a failing test that captures the bug
    ↓
fix + verify                ← agent fixes and confirms the test passes
    ↓
verification-before-completion ← agent runs full suite before claiming it's done
    ↓
finishing-a-development-branch ← PR or merge
```

**Your role:** Describe the symptom. The agent won't guess — it investigates first.

---

## 3. Code review

**Start with:** "Review this PR" or "Review what I just implemented"

```
requesting-code-review  ← agent dispatches a reviewer with full context
    ↓
[review report lands]   ← issues grouped by severity
    ↓
receiving-code-review   ← agent evaluates feedback, implements or pushes back with reasoning
```

**What you get:** A review report with critical / major / minor issues. Critical issues are fixed before anything merges. The agent doesn't blindly accept all feedback — it reasons through it.

---

## 4. Finishing a branch

**Start with:** "I'm done, let's wrap this up" or "Ready to merge"

```
verification-before-completion ← runs tests, confirms everything passes
    ↓
finishing-a-development-branch ← presents options:
                                  - Merge into main
                                  - Open a PR
                                  - Keep branch, continue later
                                  - Discard (if work was exploratory)
```

**Always run this** before considering a feature "done." It catches incomplete work and ensures the branch is clean.

---

## Combining flows

Most features use flows 1 + 4. Most bugfixes use flows 2 + 3 + 4. A typical sprint day might look like:

| Task | Flows used |
|---|---|
| Implement new API endpoint | 1 → 4 |
| Fix a reported bug | 2 → 3 → 4 |
| Review a teammate's code | 3 |
| Finish yesterday's feature | 4 |
