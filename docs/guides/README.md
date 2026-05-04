# Superpowers — Engineer's Guide

Superpowers is a Claude Code plugin that gives your AI agent a structured workflow for software development. Instead of jumping straight into code, the agent steps back, asks the right questions, writes a plan, and works through it systematically — with tests, reviews, and checkpoints built in.

**Install:**

```bash
/plugin install superpowers@claude-plugins-official
```

Then start a new Claude Code session. Skills trigger automatically — you don't need to invoke them manually.

---

## When to use Superpowers vs. plain Claude Code

Superpowers adds structured workflow overhead (brainstorm → plan → TDD → review). That overhead pays off when the work is complex or uncertain. For tasks where the approach is clear and steps are known, plain Claude Code is faster and cheaper.

**Use Superpowers when:**

- You're not sure how to approach it yet, you need brainstorm with AI
- The change touches multiple files or systems
- A mistake here would be hard to catch or costly to fix
- The task requires design decisions, tests, or a code review

**Use plain Claude Code when:**

- The approach is clear and steps are well-known
- The change is self-contained and low-risk
- You need an answer or explanation, not a code change


| Situation                                                                     | Superpowers | Plain Claude Code | Reason                                                 |
| ----------------------------------------------------------------------------- | ----------- | ----------------- | ------------------------------------------------------ |
| Bug with unknown root cause                                                   | ✅           |                   | Systematic debugging avoids wrong fixes                |
| Complex new feature (multi-system, design decisions needed)                   | ✅           |                   | Needs brainstorming, planning, TDD, and review         |
| Multi-file refactor with unclear scope                                        | ✅           |                   | Brainstorming/Planning prevents mid-refactor confusion |
| Reviewing or finishing a PR                                                   | ✅           |                   | Structured review catches gaps                         |
| Changes to auth, payments, or critical paths                                  | ✅           |                   | TDD + review reduce regression risk                    |
| New feature with clear, well-scoped requirements, step-by-step implementation |             | ✅                 | Approach is known; no planning overhead needed         |
| Bug with known root cause and clear fix steps                                 |             | ✅                 | Change is obvious; structured workflow adds no value   |
| Asking "how does X work?"                                                     |             | ✅                 | Read-only — no code change needed                      |
| Typo, rename, or single-line fix                                              |             | ✅                 | Change is obvious; no planning needed                  |
| Exploring the codebase                                                        |             | ✅                 | Read-only investigation                                |
| Adding a config value or env var                                              |             | ✅                 | Single-file, low-risk change                           |
| Explaining code to a teammate                                                 |             | ✅                 | No code change                                         |


**Quick test:** Do you know exactly what to change and how? → plain Claude Code. If there are unknowns or risks → Superpowers.

---

## Guides


| Guide                                      | What it covers                                    |
| ------------------------------------------ | ------------------------------------------------- |
| [Quick Start](quick-start.md)              | Install and complete your first task in 5 minutes |
| [Daily Workflows](daily-workflows.md)      | The 4 flows you'll use every day                  |
| [Skills Cheat Sheet](skills-cheatsheet.md) | Situation → skill → what to expect                |
| [Guide for Team Leads](team-leads.md)      | Rolling out to the team, setting norms            |


