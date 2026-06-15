# Configuring Claude Code: A Team's Cost & Safety Guide

Get your team configured correctly from day one.

---

## Table of Contents

1. [Install Claude Code](#1-install-claude-code)
2. [Configuration scopes](#2-configuration-scopes)
   - [2.1 Available Scopes](#21-available-scopes)
   - [2.2 What uses scopes](#22-what-uses-scopes)
   - [2.3 Settings File (Important)](#23-settings-file-important)
   - [2.4 What goes where (suggested practice)](#24-what-goes-where-suggested-practice)
3. [Model Selection (Important)](#3-model-selection-important)
4. [Managing Plugins and Skills](#4-managing-plugins-and-skills)
5. [Permissions](#5-permissions)
6. [Claude Code Modes](#6-claude-code-modes)
7. [Sample Session Workflow](#7-sample-session-workflow)

---

## 1. Install Claude Code

1. Native Install:
  ```bash
   curl -fsSL https://claude.ai/install.sh | bash
  ```
2. Verify the install:
  ```bash
   claude --version
  ```

Reference: [https://code.claude.com/docs/en/quickstart#native-install-recommended](https://code.claude.com/docs/en/quickstart#native-install-recommended)

---

## 2. Configuration scopes

### 2.1 Available Scopes


| Scope       | Location                                                                           | Who it affects                       | Shared with team?                           |
| ----------- | ---------------------------------------------------------------------------------- | ------------------------------------ | ------------------------------------------- |
| **Managed** | Server-managed settings, plist / registry, or system-level `managed-settings.json` | All users on the machine             | Yes (deployed by IT)                        |
| **User**    | `~/.claude/` directory                                                             | You, across all projects             | No                                          |
| **Project** | `.claude/` in repository                                                           | All collaborators on this repository | Yes (committed to git)                      |
| **Local**   | `.claude/settings.local.json`                                                      | You, in this repository only         | No (gitignored when Claude Code creates it) |


When the same setting appears in multiple scopes, Claude Code applies them in priority order (normally we use 3 scopes: Local, Project, User):

1. Managed (highest) - can’t be overridden by anything
2. Command line arguments - temporary session overrides
3. Local - overrides project and user settings (e.g .claude/setting.local.json)
4. Project - overrides user settings (e.g .claude/setting.json)
5. User (lowest) - applies when nothing else specifies the setting (e.g ~/.claude/setting.json)

---

### 2.2 What uses scopes

Scopes apply to many Claude Code features:


| Feature         | User location             | Project location                   | Local location                 |
| --------------- | ------------------------- | ---------------------------------- | ------------------------------ |
| **Settings**    | `~/.claude/settings.json` | `.claude/settings.json`            | `.claude/settings.local.json`  |
| **Subagents**   | `~/.claude/agents/`       | `.claude/agents/`                  | None                           |
| **MCP servers** | `~/.claude.json`          | `.mcp.json`                        | `~/.claude.json` (per-project) |
| **Plugins**     | `~/.claude/settings.json` | `.claude/settings.json`            | `.claude/settings.local.json`  |
| **CLAUDE.md**   | `~/.claude/CLAUDE.md`     | `CLAUDE.md` or `.claude/CLAUDE.md` | `CLAUDE.local.md`              |


On Windows, paths shown as `~/.claude` resolve to `%USERPROFILE%\.claude`.

---

### 2.3 Settings File (Important)

Settings live in JSON files at three scopes you control, plus a fourth managed by your org's IT/admin team.


| File                          | Scope   | Committed to git?   | Applies to                 |
| ----------------------------- | ------- | ------------------- | -------------------------- |
| `~/.claude/settings.json`     | User    | No                  | You, across all projects   |
| `.claude/settings.json`       | Project | Yes                 | Whole team on this project |
| `.claude/settings.local.json` | Local   | No — gitignore this | You only, on this project  |


**Precedence (highest to lowest):** Local -> Project -> User (Local setting override Project setting, which override User setting)

> A deny rule at any scope always wins — even over a more specific allow rule at a lower scope.

---

### 2.4 What goes where (suggested practice)


| Setting                               | Where it belongs                    | Why                                        |
| ------------------------------------- | ----------------------------------- | ------------------------------------------ |
| `model`                               | User `~/.claude/settings.json`      | Personal cost/performance preference       |
| `defaultMode`                         | User `~/.claude/settings.json`      | Your working style, not the team's         |
| Team plugin (`enabledPlugins`)        | Project `.claude/settings.json`     | Everyone on the team needs the same plugin |
| Shared allow list (test, lint, build) | Project `.claude/settings.json`     | Reduces prompts for all teammates          |
| Personal scripts with local paths     | Local `.claude/settings.local.json` | Paths are machine-specific                 |


> Add `.claude/settings.local.json` to your `.gitignore` immediately — it often contains local paths and personal preferences that don't belong in the repo, not shared to other team members.

#### Settings across scopes: safe default → targeted override

The table above tells you the *one* scope each setting usually lives in. But the precedence chain (Local → Project → User) lets you do something more powerful: set a safe, default at a higher scope (e.g User scope) and override it at a lower scope (e.g Local scope) only where a specific repo or task use it.
The higher scope keeps you safe everywhere; the lower scope opts you in intentionally.

**Which settings support this pattern:**


| Setting             | Safe default (higher scope) | When to override (lower scope)                 | Override scope                                           |
| ------------------- | --------------------------- | ---------------------------------------------- | -------------------------------------------------------- |
| `model`             | Sonnet at User              | Hard architecture/security task needs Opus     | **Local only** (or `/model` per-session) — never Project |
| `defaultMode`       | `"default"` at User         | Trusted, well-understood repo / fast iteration | Project or Local                                         |
| `permissions.allow` | minimal at User             | Repo-specific safe commands                    | Project (shared) / Local (personal)                      |
| `env`               | shared defaults at Project  | machine-specific values                        | Local                                                    |


> `model` and `defaultMode` are clean single-value overrides — the lowest scope that sets them wins. `permissions` instead *accumulates* across scopes, and a `deny` at any scope always beats an `allow`. `env` is replaced key-by-key, so a Local value shadows the Project value for that machine only.

##### Best practice example 1 — Cost-safe baseline, opt up only when necessary

Center the layering on `model`. Sonnet is the default everywhere; Opus is opted into per member/task, never pinned for the team.

```json
// ~/.claude/settings.json   (User — your cross-project defaults)
{
  "model": "claude-sonnet-4-6",
  "defaultMode": "default",
  "permissions": {
    "allow": [
      "Bash(git status)",
      "Bash(git log *)",
      "Bash(git diff *)"
    ]
  }
}
```

```json
// .claude/settings.json   (Project — committed, shared with the team)
{
  "model": "claude-sonnet-4-6"
}
```

```json
// .claude/settings.local.json   (Local — gitignored, yours only)
{
  "model": "claude-opus-4-8"
}
```

The override chain for `model`: Local (`claude-opus-4-8`) beats Project (`claude-sonnet-4-6`) beats User (`claude-sonnet-4-6`). Because the Local file is gitignored, the Opus upgrade — and its cost — lands only on you, for the task that needs it. Pinning Sonnet at Project keeps every teammate predictable; 
Or reach for `/model` per-session if you'd rather not persist the Opus override at all.

> **Cost Impact:** Never set `"model": "claude-opus-4-8"` in the committed `.claude/settings.json` — that pins every teammate to high cost on every request. Put Opus in `.claude/settings.local.json` (or use `/model`) when necessary, so the choice stays per-person.

##### Best practice example 2 — Cautious mode + permissions by default, loosen where trusted

Center the layering on `defaultMode` and `permissions`. Start strict (prompt on first use), then loosen only on repos you trust.

```json
// ~/.claude/settings.json   (User — safest baseline everywhere)
{
  "model": "claude-sonnet-4-6",
  "defaultMode": "default",
  "permissions": {
    "allow": [
      "Bash(git status)",
      "Bash(git log *)"
    ]
  }
}
```

```json
// .claude/settings.json   (Project — a well-understood repo, faster iteration)
{
  "defaultMode": "acceptEdits",
  "permissions": {
    "allow": [
      "Bash(npm test)",
      "Bash(npm run *)"
    ],
    "deny": [
      "Bash(git push --force *)",
      "Read(./.env)"
    ]
  },
  "env": {
    "API_BASE_URL": "https://staging.internal"
  }
}
```

```json
// .claude/settings.local.json   (Local — your machine, your risk tolerance)
{
  "defaultMode": "plan",
  "permissions": {
    "allow": [
      "Bash(~/tools/db-snapshot.sh)"
    ]
  },
  "env": {
    "API_BASE_URL": "http://localhost:8080"
  }
}
```

- `defaultMode` overrides cleanly: this machine runs in `"plan"` eventually(Local) even though the team default is `"acceptEdits"` (Project) and your personal baseline is `"default"` (User). 
- `permissions` instead *accumulates* — the Local `allow` adds to the Project and User lists rather than replacing them, and the Project `deny` on `git push --force` still wins everywhere. 
- `env` is shadowed per key: `API_BASE_URL` resolves to `localhost` locally while the team keeps `staging`. 
- Switch modes for a single session with Shift+Tab.

Reference: [https://code.claude.com/docs/en/settings](https://code.claude.com/docs/en/settings)

---

## 3. Model Selection (Important)

> **Cost Impact:** Model choice is the single largest lever on your API bill. Opus costs roughly 2x more per token than Sonnet. Haiku costs roughly one-fifth of Sonnet. Defaulting to Sonnet for all engineering work is the right baseline for most teams.

Set your model at user scope so it applies across all projects:

```json
// ~/.claude/settings.json
{
  "model": "claude-sonnet-4-6"
}
```

The shorthand alias `"sonnet"` also works. Switch models for the current session with `/model`.


| Model               | Relative cost                | When to use                                                |
| ------------------- | ---------------------------- | ---------------------------------------------------------- |
| `claude-sonnet-4-6` | 1x **(recommended default)** | All standard engineering work — coding, debugging, reviews |
| `claude-opus-4-8`   | ~2x                          | Complex architecture decisions, sensitive security reviews |
| `claude-haiku-4-5`  | ~0.2x                        | Mechanical tasks: renames, formatting, config changes      |


Reference: [https://platform.claude.com/docs/en/about-claude/pricing](https://platform.claude.com/docs/en/about-claude/pricing)

---

### Per-project model lock

Teams can pin a model in `.claude/settings.json` to keep costs predictable:

```json
// .claude/settings.json
{
  "model": "claude-sonnet-4-6"
}
```

> Don't set Opus in `.claude/settings.json` — that pins every teammate to the expensive model for every request.

---

## 4. Managing Plugins and Skills

> **Cost Impact:** Every enabled plugin injects its skill metadata into the context window on every request. Plugins you never use still consume tokens and confuse the model (worse quality). 
> -> Disable plugins your team doesn't need, check plugin/skill/mcp list at the start of every Claude session (Important)

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

> See Quick Start for how to install a plugin or register a custom plugin marketplace.

---

## 5. Permissions

Claude Code's permission system controls which tools run automatically vs. which need your approval first.


| Tool type         | Example          | Approval required | "Yes, don't ask again" behavior               |
| ----------------- | ---------------- | ----------------- | --------------------------------------------- |
| Read-only         | File reads, Grep | No                | N/A                                           |
| Bash commands     | Shell execution  | Yes               | Permanently per project directory and command |
| File modification | Edit/write files | Yes               | Until session end                             |


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


| Pattern                  | Matches                       |
| ------------------------ | ----------------------------- |
| `Bash(npm test)`         | Exactly `npm test`            |
| `Bash(npm run *)`        | Any `npm run` subcommand      |
| `Bash(git *)`            | Any git command               |
| `Bash(docker compose *)` | Any docker compose subcommand |
| `Read(~/.zshrc)`         | Reads your home `.zshrc`      |
| `Read(./.env)`           | Reads the project `.env` file |


> The space before `*` matters: `Bash(ls *)` matches `ls -la` but not `lsof`. `Bash(ls*)` without the space matches both.

### Ask lists

`ask` rules sit between `allow` and `deny`: the command still runs, but Claude prompts for confirmation every time — even when a broader `allow` rule also matches it. Use them as guardrails for actions that are fine *most* of the time but should never happen silently:

```json
// .claude/settings.json
{
  "permissions": {
    "allow": [
      "Bash(git *)"
    ],
    "ask": [
      "Bash(git push *)"
    ]
  }
}
```

Here every git command is pre-approved, but `git push` always pauses for a yes/no. Because evaluation is **deny → ask → allow**, the `ask` rule wins over the broad `git `* allow without you having to deny pushes outright.

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

### Permissions across scopes (User → Project → Local)

Like other settings, permission rules layer across scopes — but they don't *replace* each other the way `model` does. They **accumulate**, and a `deny` at any scope always wins.

**Precedence (highest to lowest):**

1. Managed (org/IT) — can't be overridden by anything, including command-line arguments
2. Command-line arguments — temporary session overrides
3. Local — `.claude/settings.local.json`
4. Project — `.claude/settings.json`
5. User — `~/.claude/settings.json`

Two rules to keep in mind:

- **Lists accumulate** — the `allow`/`ask`/`deny` arrays from every scope are merged, not overwritten. The effective `allow` set is the union of User + Project + Local.
- **Deny wins everywhere** — if a tool is denied at any level, no other level can allow it. A User-level deny blocks a Project-level allow, and a Project-level deny blocks a User- or Local-level allow.

A layered example:

```json
// ~/.claude/settings.json   (User — safe baseline everywhere)
{
  "permissions": {
    "allow": ["Bash(git status)", "Bash(git log *)", "Bash(git diff *)"],
    "deny": ["Read(./.env)", "Read(./.env.*)"]
  }
}
```

```json
// .claude/settings.json   (Project — committed, shared with the team)
{
  "permissions": {
    "allow": ["Bash(npm test)", "Bash(npm run *)"],
    "ask": ["Bash(git push *)"],
    "deny": ["Bash(git push --force *)"]
  }
}
```

```json
// .claude/settings.local.json   (Local — gitignored, yours only)
{
  "permissions": {
    "allow": ["Bash(~/tools/db-snapshot.sh)"]
  }
}
```

How this resolves:

- The effective `allow` list is the **union** of all three files — git read commands (User), npm commands (Project), and your local snapshot script (Local) all run without prompts.
- `git push` still **prompts every time** via the Project `ask` rule, even though it isn't denied.
- `git push --force` and any read of `.env`* are **blocked for everyone**, because a `deny` at any scope beats every `allow`.

### Where to put rules


| Rule type                                   | Scope                               | Reason                            |
| ------------------------------------------- | ----------------------------------- | --------------------------------- |
| Shared test/lint/build commands             | Project `.claude/settings.json`     | Reduces prompts for all teammates |
| Sensitive file deny rules (`.env`, secrets) | Project `.claude/settings.json`     | Protects everyone                 |
| Personal scripts with local paths           | Local `.claude/settings.local.json` | Paths are machine-specific        |
| Broad personal grants (e.g., all git)       | User `~/.claude/settings.json`      | Applies everywhere you work       |


Reference: [https://code.claude.com/docs/en/permissions#permission-system](https://code.claude.com/docs/en/permissions#permission-system)

---

## 6. Claude Code Modes

Modes control how autonomously Claude acts. Set `defaultMode` in your user settings and override per-project or per-session as needed.


| Mode         | `defaultMode` value   | Behavior                                                                                                                                | When to use                                                          |
| ------------ | --------------------- | --------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| Default      | `"default"`           | Standard behavior: prompts for permission on first use of each tool                                                                     | New codebases, sensitive changes, onboarding                         |
| Accept Edits | `"acceptEdits"`       | Automatically accepts file edits and common filesystem commands (`mkdir`, `touch`, `mv`, `cp`, etc.) for paths in the working directory | Trusted tasks where you want faster iteration                        |
| Plan         | `"plan"`              | Claude reads files and runs read-only shell commands to explore but does not edit your source files                                     | Complex changes where you want to review before any file is touched  |
| Auto         | `"auto"`              | Auto-approves tool calls with background safety checks that verify actions align with your request                                      | Experienced users on well-understood tasks                           |
| Don't Ask    | `"dontAsk"`           | Auto-denies tools unless pre-approved via `/permissions` or `permissions.allow` rules                                                   | Strict control — only pre-approved tools run; all others are blocked |
| Bypass       | `"bypassPermissions"` | Skips permission prompts, except those forced by explicit `ask` rules; root and home directory removals (e.g. `rm -rf /`) still prompt  | Isolated containers/VMs only — never on local dev machines           |


Set your default in user settings (User scope), then can override at Project or Local scope:

```json
// ~/.claude/settings.json
{
  "model": "claude-sonnet-4-6",
  "defaultMode": "default"
}
```

Switch modes mid-session: press Shift+Tab to cycle default -> acceptEdits -> plan -> auto. The current mode appears in the status bar. Not every mode is in the default cycle.

> Start with `"default"` and graduate to `"acceptEdits"` once you're comfortable. Use `"plan"` for any change that touches multiple files or systems.
> Use `"dontAsk"` when you want Claude to only run pre-approved tools and block everything else. 
> Only use `"bypassPermissions"` in isolated environments like CI containers or VMs — explicit `ask` rules still force a prompt even in that mode, and administrators can prevent it via `permissions.disableBypassPermissionsMode`.

> **Cost Impact:** `auto` mode issues more tool calls without pausing. Switch from `"default"` to `"auto"` only when the task scope is clear and well-bounded.

Reference: [https://code.claude.com/docs/en/permissions#permission-modes](https://code.claude.com/docs/en/permissions#permission-modes), [https://code.claude.com/docs/en/permission-modes](https://code.claude.com/docs/en/permission-modes)

---

## 7. Sample Session Workflow

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
6. Verify selected model      /model   (select suitable model)
      |
      v
7. Check MCP connections      /mcp   (disable unused ones)
      |
      v
8. Describe your task
      |
      v
9. Review plan (if plan mode) ──► Approve or revise
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
9. **Review the plan** — if in plan mode Claude shows what it intends to do before touching any files. Approve or revise.
10. **Claude executes** — monitor progress; respond to any permission prompts that arise.
11. **Inspect commits** — verify the output before pushing:
  ```bash
    git log --oneline -10
  ```

### Troubleshooting


| Symptom                                           | Likely cause                                       | Fix                                                                                                                  |
| ------------------------------------------------- | -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| Skills don't trigger                              | Plugin not enabled for this session                | Restart Claude Code after changing `enabledPlugins`; or use `/plugin` to toggle on or /reload-plugins to reload them |
| Constant permission prompts                       | Allow list missing common commands                 | Add to `permissions.allow` in `.claude/settings.json`                                                                |
| High unexpected costs                             | Model set to Opus somewhere                        | Check all three settings files; run `/model` to see current selection                                                |
| Agent acts autonomously when you expected prompts | `defaultMode` is `"auto"` or `"bypassPermissions"` | Change to `"default"` in `~/.claude/settings.json`                                                                   |
| Rules not taking effect                           | Conflicting rule at a higher scope                 | Use `/permissions` to see which file each rule comes from; remember deny always wins                                 |


---

