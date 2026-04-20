# Quick Start

Get from zero to your first AI-assisted feature in about 5 minutes.

---

## 1. Install the plugin

In a Claude Code session:

```bash
/plugin install superpowers@claude-plugins-official
```

Start a **new session** after installing — existing sessions won't pick up the plugin.

---

## 2. Tell the agent what you want to build

Open Claude Code in your project directory and describe a task naturally:

> "I want to add a search filter to the product listing page."

You don't need a detailed spec. A sentence is enough to start.

---

## 3. Answer the brainstorming questions

The agent will ask you questions one at a time to clarify what you're building — things like:
- What problem does this solve?
- Are there edge cases to handle?
- Should it work with the existing API or do we need a new endpoint?

Answer honestly. The questions are short. This usually takes 3–5 minutes.

---

## 4. Review and approve the design

Once the agent understands your intent, it presents the design in sections. Read each one and say whether it looks right. If something is off, say so — the agent will revise.

You're not locked in. This is a conversation, not a form.

---

## 5. Approve the implementation plan

After the design, the agent writes a step-by-step plan. Each step is small and testable. Review it, then say "go" (or just "looks good").

---

## 6. Watch it work

The agent executes the plan task by task:
- Writes a failing test first
- Implements the minimum code to make it pass
- Reviews its own work before moving to the next task

You can watch, intervene, or step away.

---

## What to expect

A typical first feature with Superpowers looks like:

```
You: "Add a search filter to the product listing page"
Agent: [asks 4–5 clarifying questions]
Agent: [presents design, you approve]
Agent: [presents plan, you approve]
Agent: [executes task by task, with TDD]
Agent: [asks whether to open a PR or merge]
```

The whole process — from your first message to a reviewable PR — usually takes 20–40 minutes for a medium-sized feature, with you mostly just approving checkpoints.

---

## Next step

See [Daily Workflows](daily-workflows.md) for the 4 most common patterns you'll use every day.
