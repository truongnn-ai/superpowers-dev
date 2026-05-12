# Superpowers

Superpowers is a complete software development workflow for your coding agents, built on top of a set of composable "skills" and some initial instructions that make sure your agent uses them.

## How it works

It starts from the moment you fire up your coding agent. As soon as it sees that you're building something, it *doesn't* just jump into trying to write code. Instead, it steps back and asks you what you're really trying to do. 

Once it's teased a spec out of the conversation, it shows it to you in chunks short enough to actually read and digest. 

After you've signed off on the design, your agent puts together an implementation plan that's clear enough for an enthusiastic junior engineer with poor taste, no judgement, no project context, and an aversion to testing to follow. It emphasizes true red/green TDD, YAGNI (You Aren't Gonna Need It), and DRY. 

Next up, once you say "go", it launches a *subagent-driven-development* process, having agents work through each engineering task, inspecting and reviewing their work, and continuing forward. It's not uncommon for Claude to be able to work autonomously for a couple hours at a time without deviating from the plan you put together.

There's a bunch more to it, but that's the core of the system. And because the skills trigger automatically, you don't need to do anything special. Your coding agent just has Superpowers.


## What's New

### v1.1.0 — 2026-05-12

Subagent-driven development now tiers tasks (`trivial` / `standard` / `heavy`) so simple changes skip the heavy review loop, and ITC negotiation carries explicit user-journey context between agents. Onboarding docs and install instructions also tightened up.

#### Inner-Loop Tier Model

The `writing-plans` skill now classifies every task using a [tier rubric](skills/writing-plans/tier-rubric.md). Each tier dispatches a different subagent chain in `subagent-driven-development`:

| Tier | When it applies | Subagents that run |
|---|---|---|
| `trivial` | Docs/config-only changes with no behavior change (README edits, `.gitignore`, version bumps) | implementer only |
| `standard` | Behavior-preserving refactors, renames, test-only additions | implementer → code-reviewer → test-runner |
| `heavy` | New behavior, new public API, contract changes, or anything ambiguous (default) | full pipeline: ITC negotiation → implementer → spec-reviewer → code-reviewer → test-runner |

- **Default-heavy rule** — when in doubt, classify as `heavy`. Implementers can escalate up a tier mid-task if a simple change turns out to need real review.
- **Tier reason** — every task records a `tier_reason` citing the matched rubric clause (e.g. `T1`, `S2`, `H4`), so classifications are auditable.
- **Implementer escalation exit** — `implementer-prompt.md` now has an explicit path to bail back up if the task no longer fits its assigned tier.

#### Journey Context in ITC Negotiation

ITC handshakes between `coding-agent` and `testing-agent` now exchange a structured **Journey Context** block resolved from the plan's journeys file. Each round also carries knowledge of the previous round's proposals, so 5-round negotiations actually converge instead of looping.

- `coding-agent-prompt.md` and `testing-agent-prompt.md` added Journey Context sections
- `testing-agent-prompt.md` added a journey-coverage check in its ITC review
- Negotiation prompts now explain the ITC term inline so agents don't have to chase definitions

#### Onboarding & Install Polish

- [Quick Start](docs/guides/quick-start.md) install instructions clarified for the marketplace flow
- [Guides README](docs/guides/README.md) adds a "When to use Superpowers vs plain Claude Code" decision section
- New [SDLC Mental Model](docs/guides/SDLC_mental_model_and_workflow.md) document covers the full development lifecycle with Superpowers

---

### v1.0.0 — 2026-04-25

Subagent-driven development now enforces pre-task ITC negotiation and a runtime test harness; brainstorming and planning track user journeys through a coverage matrix; and a new `docs/guides/` suite makes team onboarding straightforward.

#### ITC Negotiation and Runtime Test Harness

`subagent-driven-development` negotiates a formal **Inter-Task Contract (ITC)** before dispatching any implementer. The coding-agent and testing-agent agree on the interface, test tier, and expected outcomes *before* a line of code is written. After implementation, dedicated test-runner subagents execute unit and integration tests and must pass before the task is marked complete.

- **Solution-level ITC** — at the start of a plan run, agents negotiate a full coverage matrix across all tasks
- **5-round negotiation** — up from 3, giving agents more room to converge before escalating to the user
- **BLOCKED handling** — simple blockers resolved autonomously; complex ones surfaced to the user with full context
- **New prompt templates** — `coding-agent-prompt.md`, `testing-agent-prompt.md`, `test-runner-task-prompt.md`, `test-runner-solution-prompt.md`

#### Journey-Based Coverage Matrix

The workflow tracks **user journeys** end-to-end across skills:

- `brainstorming` enumerates journeys as a named step, producing a `<topic>-journeys.yaml`
- `writing-plans` records a `contributes_to` field per task linking it to journey IDs
- Solution ITC produces a **coverage matrix**: every journey × testing strategy must be either a runnable scenario or an explicit N/A with justification

Testing strategies are defined in `skills/subagent-driven-development/testing-strategies.md`:
`happy_path` · `negative_path` · `state_persistence` · `feature_interaction` · `auth_boundary`

#### Team Adoption Guides

A new `docs/guides/` directory provides onboarding materials for engineering teams:

| Guide | What it covers |
|---|---|
| [Quick Start](docs/guides/quick-start.md) | Install and complete your first task in 5 minutes |
| [Daily Workflows](docs/guides/daily-workflows.md) | The 4 flows you'll use every day |
| [Skills Cheat Sheet](docs/guides/skills-cheatsheet.md) | Situation → skill → what to expect |
| [Guide for Team Leads](docs/guides/team-leads.md) | Rolling out to the team, setting norms |
| [SDLC Mental Model](docs/guides/SDLC_mental_model_and_workflow.md) | Full development lifecycle with Superpowers |


## Sponsorship

If Superpowers has helped you do stuff that makes money and you are so inclined, I'd greatly appreciate it if you'd consider [sponsoring my opensource work](https://github.com/sponsors/obra).

Thanks! 

- Jesse


## Installation

**Note:** Installation differs by platform. Claude Code or Cursor have built-in plugin marketplaces. Codex and OpenCode require manual setup.

### Claude Code Official Marketplace

Superpowers is available via the [official Claude plugin marketplace](https://claude.com/plugins/superpowers)

Install the plugin from Claude marketplace:

```bash
/plugin install superpowers@claude-plugins-official
```

### Claude Code (via Plugin Marketplace)

In Claude Code, register the marketplace first:

```bash
/plugin marketplace add obra/superpowers-marketplace
```

Then install the plugin from this marketplace:

```bash
/plugin install superpowers@superpowers-marketplace
```

### Cursor (via Plugin Marketplace)

In Cursor Agent chat, install from marketplace:

```text
/add-plugin superpowers
```

or search for "superpowers" in the plugin marketplace.

### Codex

Tell Codex:

```
Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.codex/INSTALL.md
```

**Detailed docs:** [docs/README.codex.md](docs/README.codex.md)

### OpenCode

Tell OpenCode:

```
Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.opencode/INSTALL.md
```

**Detailed docs:** [docs/README.opencode.md](docs/README.opencode.md)

### Gemini CLI

```bash
gemini extensions install https://github.com/obra/superpowers
```

To update:

```bash
gemini extensions update superpowers
```

### Verify Installation

Start a new session in your chosen platform and ask for something that should trigger a skill (for example, "help me plan this feature" or "let's debug this issue"). The agent should automatically invoke the relevant superpowers skill.

## The Basic Workflow

1. **brainstorming** - Activates before writing code. Refines rough ideas through questions, explores alternatives, presents design in sections for validation. Saves design document.

2. **using-git-worktrees** - Activates after design approval. Creates isolated workspace on new branch, runs project setup, verifies clean test baseline.

3. **writing-plans** - Activates with approved design. Breaks work into bite-sized tasks (2-5 minutes each). Every task has exact file paths, complete code, verification steps.

4. **subagent-driven-development** or **executing-plans** - Activates with plan. Dispatches a tiered subagent chain per task (`trivial` skips review, `heavy` runs full ITC negotiation + spec/code review + test-runner), or executes in batches with human checkpoints.

5. **test-driven-development** - Activates during implementation. Enforces RED-GREEN-REFACTOR: write failing test, watch it fail, write minimal code, watch it pass, commit. Deletes code written before tests.

6. **requesting-code-review** - Activates between tasks. Reviews against plan, reports issues by severity. Critical issues block progress.

7. **finishing-a-development-branch** - Activates when tasks complete. Verifies tests, presents options (merge/PR/keep/discard), cleans up worktree.

**The agent checks for relevant skills before any task.** Mandatory workflows, not suggestions.

## What's Inside

### Skills Library

**Testing**
- **test-driven-development** - RED-GREEN-REFACTOR cycle (includes testing anti-patterns reference)

**Debugging**
- **systematic-debugging** - 4-phase root cause process (includes root-cause-tracing, defense-in-depth, condition-based-waiting techniques)
- **verification-before-completion** - Ensure it's actually fixed

**Collaboration** 
- **brainstorming** - Socratic design refinement
- **writing-plans** - Detailed implementation plans with task tier classification (`trivial`/`standard`/`heavy`)
- **executing-plans** - Batch execution with checkpoints
- **dispatching-parallel-agents** - Concurrent subagent workflows
- **requesting-code-review** - Pre-review checklist
- **receiving-code-review** - Responding to feedback
- **using-git-worktrees** - Parallel development branches
- **finishing-a-development-branch** - Merge/PR decision workflow
- **subagent-driven-development** - Tier-aware dispatch with ITC negotiation, runtime test harness, and journey-based coverage

**Meta**
- **writing-skills** - Create new skills following best practices (includes testing methodology)
- **using-superpowers** - Introduction to the skills system

## Philosophy

- **Test-Driven Development** - Write tests first, always
- **Systematic over ad-hoc** - Process over guessing
- **Complexity reduction** - Simplicity as primary goal
- **Evidence over claims** - Verify before declaring success

Read more: [Superpowers for Claude Code](https://blog.fsck.com/2025/10/09/superpowers/)

## Contributing

Skills live directly in this repository. To contribute:

1. Fork the repository
2. Create a branch for your skill
3. Follow the `writing-skills` skill for creating and testing new skills
4. Submit a PR

See `skills/writing-skills/SKILL.md` for the complete guide.

## Updating

Skills update automatically when you update the plugin:

```bash
/plugin update superpowers
```

## License

MIT License - see LICENSE file for details

## Community

Superpowers is built by [Jesse Vincent](https://blog.fsck.com) and the rest of the folks at [Prime Radiant](https://primeradiant.com).

For community support, questions, and sharing what you're building with Superpowers, join us on [Discord](https://discord.gg/Jd8Vphy9jq).

## Support

- **Discord**: [Join us on Discord](https://discord.gg/Jd8Vphy9jq)
- **Issues**: https://github.com/obra/superpowers/issues
- **Marketplace**: https://github.com/obra/superpowers-marketplace
