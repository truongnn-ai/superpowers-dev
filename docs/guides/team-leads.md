# Guide for Team Leads

How to roll out Superpowers to your team, set norms, and onboard new engineers.

---

## Rollout strategy

Don't deploy to the whole team at once. Use a phased approach:

**Week 1–2: Individual pilot**
- 1–2 engineers try it on a real (non-critical) feature
- Goal: understand what works, what surprises people, what breaks flow
- Collect friction points

**Week 3–4: Pair adoption**
- 3–5 engineers use it, pair on first tasks
- Document team-specific decisions (see "Norms to establish" below)
- Identify your first custom skill candidate (a workflow your team repeats often)

**Week 5+: Full team**
- All engineers use it for new features and bugfixes
- Establish code review norms (does every PR need an AI review? or only features above a certain size?)
- Run a retro after 2 sprints

---

## Norms to establish before full rollout

Decide these as a team and write them in your CLAUDE.md or a team wiki:

| Decision | Options | Recommendation |
|---|---|---|
| Where do specs live? | `docs/specs/` or alongside code | `docs/superpowers/specs/` (default) |
| Where do plans live? | `docs/plans/` or `~/.claude/plans/` | `~/.claude/plans/` for personal, `docs/plans/` for team-shared |
| Worktree directory | `~/worktrees/` or sibling of repo | Sibling of repo (`../repo-feature-name`) |
| When is AI review mandatory? | Every PR / features only / never | Features and bugfixes; skip for chores |
| Who approves the design in brainstorming? | Engineer who started it / tech lead | Engineer who started it; leads review in PR |

---

## Onboarding a new engineer

Suggested path for someone joining a Superpowers-enabled team:

**Day 1**
- Read [Quick Start](quick-start.md)
- Pair with an existing team member on one real task (observer role)
- Install the plugin, verify it triggers

**Week 1**
- Complete one feature end-to-end using the [new feature workflow](daily-workflows.md#1-building-a-new-feature)
- Complete one bugfix using the [bug fix workflow](daily-workflows.md#2-fixing-a-bug)
- Review the [Skills Cheat Sheet](skills-cheatsheet.md) — no need to memorize, just scan once

**Month 1**
- Use all 4 daily workflows at least once
- Give feedback on what's confusing or slowing them down — this improves the team's CLAUDE.md

---

## Common adoption problems

**"The agent doesn't follow the skill / jumps straight to code"**
- Check that the plugin is installed and the session was started after install
- Verify with: start a session and say "help me plan this feature" — you should see brainstorming trigger

**"Brainstorming takes too long"**
- The agent asks too many questions when the task is underspecified
- Fix: give more context upfront ("I want to add X. It should work like Y. The relevant file is Z.")

**"The plan has too many tasks / tasks are too small"**
- Normal for first-time users. The agent errs toward smaller tasks
- You can tell the agent to consolidate tasks during plan review

**"TDD is slowing us down"**
- This is usually a sign of missing test infrastructure (no test runner, no fixtures)
- Fix the test setup once; TDD becomes fast after that
- Don't disable TDD — it catches real bugs during agent runs

**"The agent did something I didn't expect"**
- Check `~/.claude/plans/` for the plan it was following
- Check git log — each task should have a commit with a clear message
- Skills are deterministic: if something unexpected happened, there's a reason in the plan or the brainstorming output
