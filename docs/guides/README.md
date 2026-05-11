# Superpowers — Engineer's Guide

Superpowers is a Claude Code plugin that gives your AI agent a structured workflow for software development. Instead of jumping straight into code, the agent steps back, asks the right questions, writes a plan, and works through it systematically — with tests, reviews, and checkpoints built in.

**Install:**
```bash
claude plugin install superpowers@claude-plugins-official
```

Then start a new Claude Code session. Skills trigger automatically — you don't need to invoke them manually.

---

## When to use Superpowers vs. plain Claude Code

| Situation | Use Superpowers | Use plain Claude Code |
|---|---|---|
| Building a new feature | ✅ | |
| Fixing a bug | ✅ | |
| Reviewing or finishing a PR | ✅ | |
| Any multi-step implementation task | ✅ | |
| Asking "how does X work?" | | ✅ |
| Quick one-liner fix (typo, rename) | | ✅ |
| Exploring the codebase | | ✅ |
| Explaining code to a teammate | | ✅ |

**Rule of thumb:** If you're about to write, change, or delete code that will be committed — use Superpowers. For everything else, plain Claude Code is fine.

---

## Guides

| Guide | What it covers |
|---|---|
| [Quick Start](quick-start.md) | Install and complete your first task in 5 minutes |
| [Daily Workflows](daily-workflows.md) | The 4 flows you'll use every day |
| [Skills Cheat Sheet](skills-cheatsheet.md) | Situation → skill → what to expect |
| [Guide for Team Leads](team-leads.md) | Rolling out to the team, setting norms |
