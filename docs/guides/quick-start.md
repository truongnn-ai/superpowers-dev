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

## 2. Configure your project

Create `.claude/settings.json` in your project root to enable the plugin for the whole team:

```json
{
  "enabledPlugins": {
    "superpowers@claude-plugins-official": true
  },
  "permissions": {
    "allow": [
      "Bash(npm test)",
      "Bash(npm run lint)"
    ]
  }
}
```

For personal overrides that stay off your team's repo, create `.claude/settings.local.json` (add it to `.gitignore`):

```json
{
  "enabledPlugins": {
    "superpowers@claude-plugins-official": false,
    "superpowers@my-custom-fork": true
  }
}
```

`settings.local.json` takes precedence over `settings.json`. Use it when you want to test a custom plugin version without affecting teammates.

---

## 3. Tell the agent what you want to build

Open Claude Code in your project directory and describe a task naturally:

> "I want to add a search filter to the product listing page."

You don't need a detailed spec. A sentence is enough to start.

> **Note on triggering skills:** Skills fire automatically from natural language. If the agent doesn't pick up the right skill, name it explicitly:
> - *"use the brainstorming skill"*
> - *"use superpowers:brainstorming"*
>
> Explicit invocation is the safe way to guarantee exactly the skill you want fires.

---

## 4. Answer the brainstorming questions

The agent will ask you questions one at a time to clarify what you're building — things like:
- What problem does this solve?
- Are there edge cases to handle?
- Should it work with the existing API or do we need a new endpoint?

Answer honestly. The questions are short. This usually takes 3–5 minutes.

---

## 5. Review and approve the design

Once the agent understands your intent, it presents the design in sections. Read each one and say whether it looks right. If something is off, say so — the agent will revise.

You're not locked in. This is a conversation, not a form.

---

## 6. Approve the implementation plan

After the design, the agent writes a step-by-step plan. Each step is small and testable. Review it, then say "go" (or just "looks good").

---

## 7. Watch it work

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

## Using a customized plugin

If your team maintains a fork of Superpowers (with custom skills or workflow tweaks), install it instead of the official version:

**1. Fork and clone the plugin**

```bash
git clone https://github.com/YOUR-ORG/superpowers.git
```

**2. Register your fork as a marketplace**

```bash
/plugin marketplace add YOUR-ORG/superpowers-marketplace
```

**3. Install from your marketplace**

```bash
/plugin install superpowers@YOUR-ORG-superpowers-marketplace
```

**4. Update `.claude/settings.json` to use your version**

```json
{
  "enabledPlugins": {
    "superpowers@claude-plugins-official": false,
    "superpowers@YOUR-ORG-superpowers-marketplace": true
  }
}
```

**5. Restart your Claude Code session**

To switch back to the official plugin, reverse the `true`/`false` values in `settings.json`.

> **Tip:** If only you (not the whole team) should use the custom plugin, put this config in `.claude/settings.local.json` instead.

---

## Next step

See [Daily Workflows](daily-workflows.md) for the 4 most common patterns you'll use every day.
