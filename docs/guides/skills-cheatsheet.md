# Skills Cheat Sheet

Quick reference: situation → skill → what the agent does.

You never invoke skills manually. This table helps you recognize what's happening and why.

---

## Trigger → Skill → Outcome

| When you say / do this... | Skill that fires | What to expect |
|---|---|---|
| "I want to build / add X" | `brainstorming` | Agent asks clarifying questions one at a time, produces a design doc |
| Design is approved | `writing-plans` | Agent breaks the work into bite-sized tasks with exact file paths and test steps |
| Plan is approved | `using-git-worktrees` | Agent creates an isolated branch so your main branch stays clean |
| Plan execution begins | `subagent-driven-development` | Agent runs each task with a fresh subagent, reviews the work, loops until done |
| Writing any code | `test-driven-development` | Agent writes a failing test first, then the minimum code to pass it — always |
| "There's a bug" / test fails | `systematic-debugging` | Agent investigates root cause before proposing any fix |
| About to say "it's done" | `verification-before-completion` | Agent runs tests fresh and confirms output before claiming success |
| "Review this" / before merging | `requesting-code-review` | Agent dispatches a reviewer subagent with full context, returns a severity-ranked report |
| Code review feedback arrives | `receiving-code-review` | Agent reads, verifies, and reasons through feedback — doesn't blindly accept everything |
| "Let's wrap up / merge / PR" | `finishing-a-development-branch` | Agent verifies tests, presents merge / PR / keep / discard options |
| 2+ independent tasks exist | `dispatching-parallel-agents` | Agent splits tasks across parallel subagents for speed |
| Any new session starts | `using-superpowers` | Agent checks available skills before doing anything |

---

## Skills you'll see most often

As a fullstack engineer, day-to-day you'll mostly encounter:

1. `brainstorming` — every new feature starts here
2. `test-driven-development` — fires on every implementation task
3. `systematic-debugging` — fires on every bug or failing test
4. `verification-before-completion` — fires before any "done" claim
5. `finishing-a-development-branch` — fires at the end of every piece of work

The others (`using-git-worktrees`, `subagent-driven-development`, `requesting-code-review`, `receiving-code-review`) are part of the flow but require less active involvement from you.

---

## Skills you'll rarely need to think about

| Skill | When it matters |
|---|---|
| `writing-plans` | Runs automatically after brainstorming |
| `executing-plans` | Alternative to subagent-driven-development for simpler tasks |
| `dispatching-parallel-agents` | Large features with clearly independent workstreams |
| `writing-skills` | Only if your team wants to create custom skills |
