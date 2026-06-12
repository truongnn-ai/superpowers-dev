# Claude Code Settings for Engineering Teams

Get your team configured correctly from day one.

---

## Install Claude Code

1. Install the CLI via npm:

   ```bash
   npm install -g @anthropic-ai/claude-code
   ```

2. Verify the install:

   ```bash
   claude --version
   ```

> Start a new terminal session after install — the `claude` command won't be on PATH until the shell re-reads the npm global bin directory.

---

## Settings Scopes

Settings live in JSON files at three scopes you control, plus a fourth managed by your org's IT/admin team.

| File | Scope | Committed to git? | Applies to |
| --- | --- | --- | --- |
| `~/.claude/settings.json` | User | No | You, across all projects |
| `.claude/settings.json` | Project | Yes | Whole team on this project |
| `.claude/settings.local.json` | Local | No — gitignore this | You only, on this project |
| System / MDM | Managed | N/A | Org-wide policy, cannot be overridden |

**Precedence (highest to lowest):** Managed → Local → Project → User

> A deny rule at any scope always wins — even over a more specific allow rule at a lower scope. Local settings override project, which override user.

### What goes where

| Setting | Where it belongs | Why |
| --- | --- | --- |
| `model` | User `~/.claude/settings.json` | Personal cost/performance preference |
| `defaultMode` | User `~/.claude/settings.json` | Your working style, not the team's |
| Team plugin (`enabledPlugins`) | Project `.claude/settings.json` | Everyone on the team needs the same plugin |
| Shared allow list (test, lint, build) | Project `.claude/settings.json` | Reduces prompts for all teammates |
| Personal scripts with local paths | Local `.claude/settings.local.json` | Paths are machine-specific |

> Add `.claude/settings.local.json` to your `.gitignore` immediately — it often contains local paths and personal preferences that don't belong in the repo.

**Project settings scaffold** (commit this):

```json
// .claude/settings.json
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

**Local settings scaffold** (gitignore this):

```json
// .claude/settings.local.json
{
  "model": "claude-sonnet-4-6",
  "defaultMode": "default"
}
```

---

## Model Selection

> ⚠️ **Cost Impact:** Model choice is the single largest lever on your API bill. Opus costs roughly 5× more per token than Sonnet. Haiku costs roughly one-fifth of Sonnet. Defaulting to Sonnet for all engineering work is the right baseline for most teams.

Set your model at user scope so it applies across all projects:

```json
// ~/.claude/settings.json
{
  "model": "claude-sonnet-4-6"
}
```

The shorthand alias `"sonnet"` also works. Switch models for the current session with `/model`.

| Model | Relative cost | When to use |
| --- | --- | --- |
| `claude-sonnet-4-6` | 1× **(recommended default)** | All standard engineering work — coding, debugging, reviews |
| `claude-opus-4-8` | ~5× | Complex architecture decisions, sensitive security reviews |
| `claude-haiku-4-5` | ~0.2× | Mechanical tasks: renames, formatting, config changes |

### Per-project model lock

Teams can pin a model in `.claude/settings.json` to keep costs predictable:

```json
// .claude/settings.json
{
  "model": "claude-sonnet-4-6",
  "enabledPlugins": {
    "superpowers@claude-plugins-official": true
  }
}
```

> Don't set Opus in `.claude/settings.json` — that pins every teammate to the expensive model for every request.

---

## Managing Plugins and Skills

> ⚠️ **Cost Impact:** Every enabled plugin injects its skill metadata into the context window on every request. Plugins you never use still consume tokens. Disable plugins your team doesn't need.

### Method 1 — settings file (permanent)

Edit `enabledPlugins` in `.claude/settings.json`. This applies to all future sessions for the team:

```json
// .claude/settings.json
{
  "enabledPlugins": {
    "superpowers@claude-plugins-official": true,
    "unused-plugin@marketplace": false
  }
}
```

Setting a plugin to `false` removes its metadata from the context window entirely. The plugin remains installed but costs zero tokens per request. **Restart Claude Code for changes to take effect.**

### Method 2 — interactive `/plugin` command (current session)

Type `/plugin` inside a running Claude Code session. An interactive selector appears — use arrow keys to toggle plugins and skills on or off, then confirm. Changes apply immediately to the current session and are persisted to settings.

> See [Quick Start](quick-start.md) for how to install a plugin or register a custom plugin marketplace.

---

## Permissions

Claude Code's permission system controls which tools run automatically vs. which need your approval first.

| Tool type | Default behavior |
| --- | --- |
| Read-only (file reads, grep, git log) | No prompt required |
| Bash commands | Prompts on first use per project |
| File edits | Prompts until session ends |

**Rules evaluate in this order: deny → ask → allow.** A matching deny rule always wins.

Manage rules interactively with `/permissions` — it lists all active rules and which settings file each one comes from.

### Allow lists

Add commands your team runs frequently to avoid repetitive prompts:

```json
// .claude/settings.json
{
  "permissions": {
    "allow": [
      "Bash(npm test)",
      "Bash(npm run *)",
      "Bash(npm run build)",
      "Bash(git status)",
      "Bash(git log *)",
      "Bash(git diff *)"
    ]
  }
}
```

**Wildcard patterns:**

| Pattern | Matches |
| --- | --- |
| `Bash(npm test)` | Exactly `npm test` |
| `Bash(npm run *)` | Any `npm run` subcommand |
| `Bash(git *)` | Any git command |
| `Bash(docker compose *)` | Any docker compose subcommand |
| `Read(~/.zshrc)` | Reads your home `.zshrc` |
| `Read(./.env)` | Reads the project `.env` file |

> The space before `*` matters: `Bash(ls *)` matches `ls -la` but not `lsof`. `Bash(ls*)` without the space matches both.

### Deny lists

Block operations your team should never need Claude to run:

```json
// .claude/settings.json
{
  "permissions": {
    "deny": [
      "Bash(git push --force *)",
      "Read(./.env)",
      "Read(./.env.*)"
    ]
  }
}
```

> Never add `Bash(*)` to project settings — that grants Claude unrestricted shell execution for every teammate.

### Where to put rules

| Rule type | Scope | Reason |
| --- | --- | --- |
| Shared test/lint/build commands | Project `.claude/settings.json` | Reduces prompts for all teammates |
| Sensitive file deny rules (`.env`, secrets) | Project `.claude/settings.json` | Protects everyone |
| Personal scripts with local paths | Local `.claude/settings.local.json` | Paths are machine-specific |
| Broad personal grants (e.g., all git) | User `~/.claude/settings.json` | Applies everywhere you work |

---

## Claude Code Modes

Modes control how autonomously Claude acts. Set `defaultMode` in your user settings and override per-project or per-session as needed.

| Mode | `defaultMode` value | Behavior | When to use |
| --- | --- | --- | --- |
| Default | `"default"` | Prompts for permission on first use of each tool | New codebases, sensitive changes, onboarding |
| Accept Edits | `"acceptEdits"` | Auto-accepts file edits and common filesystem ops | Trusted tasks where you want faster iteration |
| Plan | `"plan"` | Reads files and explores, but does not edit — pauses to show a plan first | Complex changes where you want to review before any file is touched |
| Auto | `"auto"` | Auto-approves most tool calls with background safety checks *(research preview)* | Experienced users on well-understood tasks |
| Bypass | `"bypassPermissions"` | Skips all permission prompts | Isolated containers/VMs only — never on local dev machines |

Set your default in user settings:

```json
// ~/.claude/settings.json
{
  "model": "claude-sonnet-4-6",
  "defaultMode": "default"
}
```

Switch modes mid-session: `/mode acceptEdits`, `/mode plan`, `/mode auto`.

> Start with `"default"` and graduate to `"acceptEdits"` once you're comfortable. Use `"plan"` for any change that touches multiple files or systems. Only use `"bypassPermissions"` in isolated environments like CI containers or VMs.

> ⚠️ **Cost Impact:** `auto` mode issues more tool calls without pausing. On long multi-step tasks the token difference compounds. Switch from `"default"` to `"auto"` only when the task scope is clear and well-bounded.

> For Superpowers-driven workflows the skill manages mode transitions internally. See [Daily Workflows](daily-workflows.md) for how tasks progress through plan and execute phases.

---

## Sample Session Workflow

This is the recommended checklist for starting a new Claude Code session. Run through it once until the steps are automatic.

```
Start session
      |
      v
1. Verify global settings     cat ~/.claude/settings.json
      |
      v
2. Confirm project plugin     cat .claude/settings.json
      |
      v
3. Check gitignore            grep settings.local .gitignore
      |
      v
4. Launch Claude Code         claude
      |
      v
5. Confirm plugins & skills   /plugin   (disable unused ones)
      |
      v
6. Verify selected model      /model
      |
      v
7. Check MCP connections      /mcp
      |
      v
8. Describe your task
      |
      v
9. Review plan (plan mode) ──► Approve or revise
      |
      v
10. Claude executes
      |
      v
11. Inspect commits           git log --oneline -10
```

**Steps in detail:**

1. **Verify global settings** — confirm your model and defaultMode are set:
   ```bash
   cat ~/.claude/settings.json
   ```

2. **Confirm project plugin** — check the team plugin is enabled:
   ```bash
   cat .claude/settings.json
   ```

3. **Check gitignore** — make sure local settings won't be committed:
   ```bash
   grep settings.local .gitignore
   ```
   If nothing prints, add `.claude/settings.local.json` to `.gitignore`.

4. **Launch Claude Code** from the project root:
   ```bash
   claude
   ```

5. **Confirm plugins and skills** — type `/plugin` and disable any plugins you won't use this session.

6. **Verify your model** — type `/model` to confirm the active model. Change it here if needed for this session.

7. **Check MCP connections** — type `/mcp` to see which MCP servers are active. Disconnect any you don't need for this task.

8. **Describe your task** — use natural language. If you're unsure of the approach, start with brainstorming; the Superpowers skill triggers automatically.

9. **Review the plan** — in plan mode Claude shows what it intends to do before touching any files. Approve or revise.

10. **Claude executes** — monitor progress; respond to any permission prompts that arise.

11. **Inspect commits** — verify the output before pushing:
    ```bash
    git log --oneline -10
    ```

### Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Skills don't trigger | Plugin not enabled for this session | Restart Claude Code after changing `enabledPlugins`; or use `/plugin` to toggle on |
| Constant permission prompts | Allow list missing common commands | Add to `permissions.allow` in `.claude/settings.json` |
| High unexpected costs | Model set to Opus somewhere | Check all three settings files; run `/model` to see current selection |
| Agent acts autonomously when you expected prompts | `defaultMode` is `"auto"` or `"bypassPermissions"` | Change to `"default"` in `~/.claude/settings.json` |
| Rules not taking effect | Conflicting rule at a higher scope | Use `/permissions` to see which file each rule comes from; remember deny always wins |

---

> For team rollout strategy and shared norms, see [Guide for Team Leads](team-leads.md).
> For the 4 daily workflows you'll repeat most often, see [Daily Workflows](daily-workflows.md).
