# Memory Layer Design

> A file-based, git-trackable, human-readable persistent memory system for the superpowers workflow.

## Problem

The superpowers workflow is deliberately stateless. Each session starts fresh, each subagent gets isolated context. Persistence happens through design specs, implementation plans, and git commits. But institutional knowledge — why decisions were made, what errors were solved, user preferences, architecture context — is lost between sessions. Every new session restarts the learning curve.

## Design Constraints

- **File-based** — markdown files, not a database
- **Git-managed** — history, blame, diffs, PRs all work naturally
- **Human-readable** — users can browse, edit, review memories directly
- **Per-project** — memory lives inside each repo, not globally
- **Agent-driven discovery** — agent decides what to read based on the current task, not hardcoded per skill

## Research Foundation

This design draws from:
- **ByteRover Context Tree** — hierarchical domain/topic/facts structure stored as markdown in `.brv/context-tree/`. Achieved 96.1% accuracy on LoCoMo benchmark. Key insight: pre-organizing knowledge before queries arrive is the key to retrieval accuracy.
- **ADR (Architecture Decision Records)** — append-only decisions with supersession links. Sequential numbering, never mutate old records.
- **Zettelkasten / Obsidian MOC pattern** — atomic notes (one concept per file), Maps of Content as curated indices when topics grow large.
- **Claude Code Auto-Memory** — MEMORY.md as index (200-line cap), topic files loaded on demand. Auto-Dream consolidation (24h + 5 sessions trigger).
- **OpenClaw flush-before-forget** — persist important observations before context compaction.
- **TIL repos** — two-level max hierarchy, auto-generated README index.

## Architecture

### Approach: Flat Categories + Domain Tags in Frontmatter

Categories provide the primary folder structure (what kind of memory). Domain tags in frontmatter enable cross-cutting queries (what domain it relates to). INDEX.md provides two views — by category and by domain — giving agents two entry points for discovery.

### Memory Categories

| Category | Purpose | Examples |
|----------|---------|---------|
| `decisions` | Why we chose X over Y | "PostgreSQL over MongoDB for ACID compliance" |
| `preferences` | User workflow/style preferences | "Single bundled PR for refactors" |
| `errors` | Error patterns and their fixes | "Docker port 5432 conflict — stop local postgres" |
| `architecture` | System context and constraints | "Auth rewrite driven by compliance, not tech debt" |
| `learnings` | Agent/workflow observations | "Haiku model fails on 100+ line refactors" |

## Memory File Format

Each memory is a markdown file with YAML frontmatter:

```markdown
---
title: "Use PostgreSQL over MongoDB for session storage"
type: decision
tags: [auth, database]
created: 2026-03-15
source: auto:brainstorming
supersedes: decisions/2026-03-01-mongodb-session-storage.md
confidence: high
---

## Context
During the auth service design, we evaluated MongoDB vs PostgreSQL for session token storage.

## Decision
PostgreSQL, because the compliance team requires ACID transactions for session data and we already run Postgres for the user table.

## Consequences
- Session queries use SQL joins instead of document lookups
- Need connection pooling (pgbouncer) for high-concurrency scenarios
```

### Frontmatter Fields

| Field | Required | Values | Purpose |
|-------|----------|--------|---------|
| `title` | yes | string | Short descriptive title |
| `type` | yes | `decision` \| `preference` \| `error` \| `architecture` \| `learning` | Memory category |
| `tags` | yes | string array | Domain tags for cross-cutting discovery |
| `created` | yes | YYYY-MM-DD | When the memory was created |
| `source` | yes | `auto:{skill}` \| `manual` | How it was captured |
| `supersedes` | no | relative path | ADR-style link to the memory this replaces |
| `confidence` | no | `high` \| `medium` \| `low` | Agent can deprioritize low-confidence entries |
| `merged_from` | no | path array | Links to original memories before consolidation |

### Body Structure by Type

- **Decisions**: Context / Decision / Consequences
- **Errors**: Symptoms / Root Cause / Fix
- **Preferences**: freeform
- **Architecture**: freeform, describe current state and constraints
- **Learnings**: freeform, describe what was observed and what to do differently

### File Naming

- Dated entries (decisions, errors): `{category}/{YYYY-MM-DD}-{slug}.md`
- Undated entries (preferences, architecture): `{category}/{slug}.md`
- Slugs are lowercase, hyphen-separated, descriptive

## INDEX.md Structure

INDEX.md is auto-generated and injected into agent context at session start. It provides two views:

```markdown
# Memory Index

> Auto-generated. Do not edit manually.
> Last updated: 2026-04-03T14:30:00Z
> Total memories: 12

## By Category

### Decisions (4)
- [Use PostgreSQL for session storage](decisions/2026-03-15-postgres-over-mongodb.md) — ACID compliance for auth tokens `[auth, database]`
- [Server components architecture](decisions/2026-03-28-server-components.md) — RSC for data-heavy pages `[frontend]`
- [Monorepo structure](decisions/2026-04-01-monorepo-structure.md) — turborepo over nx `[infrastructure]` *(superseded)*
- [Pnpm workspaces](decisions/2026-04-02-pnpm-workspaces.md) — supersedes monorepo-structure `[infrastructure]`

### Preferences (2)
- [PR style](preferences/pr-style.md) — single bundled PR for refactors `[workflow]`
- [Testing approach](preferences/testing-approach.md) — TDD for APIs, skip for scripts `[workflow, testing]`

### Errors (3)
- [Docker port conflict on 5432](errors/2026-03-20-docker-port-conflict.md) — stop local postgres first `[infrastructure, docker]`
- [JWT expiry race condition](errors/2026-03-25-jwt-expiry-race.md) — add 5s clock skew tolerance `[auth]`
- [Legacy peer deps](errors/2026-03-31-legacy-peer-deps.md) — npm needs --legacy-peer-deps `[frontend, dependencies]`

### Architecture (2)
- [Auth service rewrite](architecture/auth-service-rewrite.md) — driven by compliance, not tech debt `[auth, compliance]`
- [API gateway pattern](architecture/api-gateway.md) — Kong for rate limiting + routing `[infrastructure, api]`

### Learnings (1)
- [Haiku fails complex refactors](learnings/haiku-model-complex-refactors.md) — use sonnet for 100+ line changes `[agent, subagent]`

## By Domain

### auth (3)
- [Use PostgreSQL for session storage](decisions/2026-03-15-postgres-over-mongodb.md)
- [JWT expiry race condition](errors/2026-03-25-jwt-expiry-race.md)
- [Auth service rewrite](architecture/auth-service-rewrite.md)

### infrastructure (4)
- [Docker port conflict](errors/2026-03-20-docker-port-conflict.md)
- [Monorepo structure](decisions/2026-04-01-monorepo-structure.md) *(superseded)*
- [Pnpm workspaces](decisions/2026-04-02-pnpm-workspaces.md)
- [API gateway pattern](architecture/api-gateway.md)

### frontend (2)
- [Server components](decisions/2026-03-28-server-components.md)
- [Legacy peer deps](errors/2026-03-31-legacy-peer-deps.md)
```

### INDEX.md Design Principles

- **Two views, same data** — agent scans by "what kind" (category) or "what about" (domain)
- **One-liner summaries** — enough context to decide whether to read the full file
- **Superseded entries marked** — visible but clearly outdated
- **Tags inline** — agent can grep for a domain without parsing frontmatter
- **Under 200 lines** — fits in context window budget

## Automatic Capture Points

Skills auto-write memories at key workflow moments:

| Skill | Trigger | Memory Type |
|-------|---------|-------------|
| `brainstorming` | User approves a design decision | `decision` |
| `brainstorming` | User states a preference during Q&A | `preference` |
| `systematic-debugging` | Root cause identified and fix verified | `error` |
| `docker-verified-execution` | Ralph Loop hits same error 2+ times then resolves | `error` |
| `writing-plans` | Architecture constraints captured in plan | `architecture` |
| `subagent-driven-development` | Subagent model/approach fails or succeeds unexpectedly | `learning` |
| `requesting-code-review` | Reviewer flags a pattern to remember | `learning` |

**Manual capture** works anytime — the agent writes a memory when asked ("remember this") or when it judges something worth persisting.

**Capture heuristic:** "Would a new team member benefit from knowing this?" If the answer is no, skip it. Routine decisions and common errors don't need memory entries.

## Skills Architecture

Three skills form the memory layer:

### Skill 1: `memory-capture`

**Invoked by:** other skills at capture moments, or manually by user.

**Responsibilities:**
1. Create the memory file with correct frontmatter and naming
2. Place in the correct category folder
3. Dedup check — grep existing memories for similar title/content, skip if >85% overlap
4. Regenerate INDEX.md after write

**Inputs from invoking skill:**
- `type` — decision | preference | error | architecture | learning
- `title` — short descriptive title
- `tags` — domain tags array
- `context` — what happened (the skill passes its relevant context)
- `source` — auto:{skill-name} | manual
- `supersedes` — path to old memory (optional)

**Integration pattern:** Each skill that captures memories adds a one-line invocation — "invoke memory-capture with type X and context Y." The memory-capture skill handles all mechanics.

### Skill 2: `memory-consolidate`

**Invoked:** manually ("consolidate memories"), or recommended when INDEX.md exceeds 150 entries.

**Four phases:**

**Phase 1 — Scan:**
- Read all memory files and INDEX.md
- Group by category and domain tags
- Flag candidates: multiple errors with same root cause, decisions superseded 2+ times, duplicate learnings

**Phase 2 — Merge:**
- Combine related memories into a single richer memory
- Example: three Docker port errors become one "Docker networking gotchas" memory
- Set `merged_from: [path1, path2, path3]` in frontmatter
- Delete original files (git history preserves them)

**Phase 3 — Prune:**
- Remove memories no longer valid:
  - Superseded decision chains — delete all but the leaf
  - Error memories whose referenced fix is now in the codebase
  - Architecture memories describing removed components
- Validate references — if a memory mentions a file path, check it exists. Flag stale.

**Phase 4 — Rebuild:**
- Regenerate INDEX.md from surviving memories
- Git commit with descriptive message
- Report what was merged/pruned (user reviews via `git diff`)

**Key principle:** every merge/prune is a git commit. User can `git revert` if consolidation was too aggressive.

### Skill 3: `memory-recall` (instruction block, not a full skill)

**Referenced by:** any skill that needs context before starting work.

**How it works:**
1. INDEX.md is already in context (injected at startup)
2. Agent matches domain tags and categories to the current task
3. Agent reads 2-5 relevant memory files
4. Agent proceeds with recalled context

This is a shared instruction block (`@memory-shared/recall-instructions.md`) referenced by other skills, not a standalone skill invocation. Keeps it lightweight.

## INDEX.md Regeneration & Growth Management

### Regeneration

Triggered after every `memory-capture` write and every `memory-consolidate` run. The process:
1. Glob `docs/superpowers/memory/**/*.md` (excluding INDEX.md)
2. Parse YAML frontmatter from each file
3. Sort by category, then by date descending
4. Build "By Domain" section by grouping across all tags
5. Write INDEX.md

### Growth Guardrails

| Threshold | Action |
|-----------|--------|
| < 50 memories | Everything indexed. No intervention needed. |
| 50-150 memories | Organic subfolders appear (e.g., `errors/docker/`). INDEX.md still lists everything. |
| 150+ memories | `memory-consolidate` recommended. Superseded entries pruned from index. |
| 200+ lines in INDEX.md | Older superseded and low-confidence entries drop from index. An `## Archived` section lists paths-only for entries still on disk but not actively indexed. |

### What stays vs. what gets archived from INDEX.md

- **Always indexed:** active decisions (not superseded), current preferences, recent errors (< 6 months), all architecture entries
- **Archived from index (file stays):** superseded decisions, resolved errors older than 6 months, learnings consolidated into broader entries
- **Deleted (git preserves):** memories whose referenced code/files no longer exist, after `memory-consolidate` validates

## SessionStart Hook Integration

### How memories enter agent context

The existing SessionStart hook (`hooks/session-start`) already injects `using-superpowers/SKILL.md`. We extend it to also inject `docs/superpowers/memory/INDEX.md`.

```
Session starts
  └─ SessionStart hook fires
      ├─ Injects using-superpowers/SKILL.md (existing)
      └─ Injects docs/superpowers/memory/INDEX.md (new)
```

### Agent discovery flow

1. Agent receives task from user
2. INDEX.md is already in context
3. Agent scans INDEX.md — matches domain tags and categories to the task
4. Agent reads 2-5 relevant memory files (full content)
5. Agent proceeds with the task, informed by recalled context

### Addition to `using-superpowers` skill

```markdown
## Memory Layer
Your project has a persistent memory at `docs/superpowers/memory/`.
INDEX.md is loaded into your context at session start. Before starting
any task, scan the index for memories relevant to your current work —
match by domain tags and category. Read the full files for anything
that looks relevant. Do not load all memories — only what the task needs.

When you make a significant decision, solve a hard bug, or learn
something surprising, invoke the memory-capture skill to persist it.
```

### Resume sessions

The hook fires on `startup|clear|compact` but not `--resume`. For resume sessions, INDEX.md isn't re-injected, but skills' recall instructions can read it on demand. No hook trigger change needed.

## Complete File Tree

```
docs/superpowers/memory/
├── INDEX.md                          # auto-generated, injected at startup
├── decisions/
│   ├── 2026-03-15-postgres-over-mongodb.md
│   ├── 2026-03-28-server-components.md
│   └── 2026-04-02-pnpm-workspaces.md
├── preferences/
│   ├── pr-style.md
│   └── testing-approach.md
├── errors/
│   ├── 2026-03-20-docker-port-conflict.md
│   ├── 2026-03-25-jwt-expiry-race.md
│   └── docker/                        # organic subfolder when 10+ entries
│       ├── 2026-03-31-compose-watch-race.md
│       └── 2026-04-01-healthcheck-timing.md
├── architecture/
│   ├── auth-service-rewrite.md
│   └── api-gateway.md
└── learnings/
    ├── haiku-model-complex-refactors.md
    └── subagent-context-crafting.md

skills/
├── memory-capture/
│   └── SKILL.md
├── memory-consolidate/
│   └── SKILL.md
├── memory-shared/
│   ├── capture-instructions.md        # shared format instructions, @imported by skills
│   └── recall-instructions.md         # shared recall instructions, @imported by skills
└── [existing skills with capture triggers added]
    ├── brainstorming/SKILL.md
    ├── systematic-debugging/SKILL.md
    ├── docker-verified-execution/SKILL.md
    ├── writing-plans/SKILL.md
    ├── subagent-driven-development/SKILL.md
    ├── requesting-code-review/SKILL.md
    └── using-superpowers/SKILL.md
```

## Workflow Walkthrough

### Session 1 — Building an auth service

1. User: "Build a login system"
2. `brainstorming` activates, agent scans INDEX.md
3. Finds `architecture/auth-service-rewrite.md` (tags: [auth, compliance]) and `decisions/2026-03-15-postgres-over-mongodb.md` (tags: [auth, database])
4. Reads both — now knows auth is compliance-driven and uses Postgres
5. Asks better-informed design questions
6. User approves design — brainstorming invokes `memory-capture`: type=decision, title="OAuth2 + PKCE for login flow", tags=[auth, security]
7. Memory file written, INDEX.md rebuilt

### Session 2 — Debugging a related issue

1. User: "Login tokens expire too fast"
2. `systematic-debugging` activates, agent scans INDEX.md
3. Finds `errors/2026-03-25-jwt-expiry-race.md` (tags: [auth]) — reads it
4. Prior error was clock skew; checks if same symptom — different root cause this time
5. Fix verified — invokes `memory-capture`: type=error, title="Refresh token TTL must match access token TTL", tags=[auth, security]

### Session 5 — Consolidation

1. User: "consolidate memories"
2. `memory-consolidate` activates
3. Scan: finds 3 auth error memories with related root causes
4. Merge: combines into "Auth token lifecycle gotchas" with `merged_from` links
5. Prune: removes superseded decision from index
6. Rebuild: new INDEX.md, git commit with descriptive message
7. User reviews: `git diff HEAD~1 docs/superpowers/memory/`
