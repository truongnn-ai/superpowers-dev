# Docker-Verified Figma E2E Pipeline — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend the superpowers skill pipeline with Docker-first verified execution, Figma MCP design extraction, configurable verification profiles, and comprehensive Playwright e2e testing.

**Architecture:** Layered extension — 4 existing skills get targeted modifications, 4 new skills are created. All skills are pure Markdown (no runtime code). Skills live in the user's project `.claude/skills/` directory. One 3-line edit to superpowers' `brainstorming/SKILL.md` checklist connects everything.

**Tech Stack:** Markdown skill files, YAML frontmatter, Figma MCP (cursor-talk-to-figma-mcp), Playwright, Docker Compose, jq

**Spec:** `docs/superpowers/specs/2026-03-31-docker-verified-figma-e2e-pipeline-design.md`

---

## File Structure

### Files to Modify

| File | Responsibility |
|---|---|
| `.claude/skills/verification-gate/SKILL.md` | Add per-task mode, Signal 6 (E2E), assume Docker running |
| `.claude/skills/dst-simulation/SKILL.md` | Add "reuse running containers" mode |
| `.claude/skills/docker-patterns/SKILL.md` | Add Task 0 environment setup workflow |
| `.claude/skills/brainstorming/SKILL.md` | Add Figma question to checklist (3 lines) |

### Files to Create

| File | Responsibility |
|---|---|
| `.claude/skills/figma-mcp-guide/SKILL.md` | Figma MCP usage: connect, extract tokens, map components |
| `.claude/skills/docker-verified-execution/SKILL.md` | Ralph Loop: deploy → test → logs → debug → retry |
| `.claude/skills/verification-profiles/SKILL.md` | Task-type → verification signal mapping |
| `.claude/skills/e2e-verification/SKILL.md` | Final Playwright gate: Figma compliance + business workflows |

All file paths below are relative to `/Users/nguyenviethung/auto-blog-verify-gate/`.

---

## Task 1: Create `verification-profiles` Skill

This is a dependency for other skills, so it comes first.

**Files:**
- Create: `.claude/skills/verification-profiles/SKILL.md`

- [ ] **Step 1: Create the skill file**

```markdown
---
name: verification-profiles
description: "Use when writing implementation plans that include verification — defines which verification signals each task type requires based on file paths and task category"
---

# Verification Profiles

Map task types to verification signals so that each task runs only the checks it needs.

## When to Use

During plan writing (writing-plans skill). Tag each task in the plan with a verification profile. The profile determines which signals `verification-gate` runs for that task.

## Profile Definitions

| Profile | Signals | Playwright Project | Pytest Marker | When to Apply |
|---|---|---|---|---|
| `ui-component` | containers, lint, e2e | Figma-Compliance | — | CSS/layout changes, new UI components |
| `api-endpoint` | containers, lint, property, contract | — | `api` | REST/GraphQL endpoint changes |
| `integration` | containers, lint, e2e, contract | Business-Workflows | `integration` | Changes spanning frontend + backend |
| `database` | containers, lint, property, contract | — | `db` | Schema migrations, query changes |
| `infrastructure` | containers, lint | — | `smoke` | Docker, CI/CD, config changes |
| `refactoring` | containers, lint, property, e2e, contract | Business-Workflows | — (run all) | Restructuring without behavior change |

## Profile Selection Rules

1. Parse task description + file paths in the plan
2. Match against profiles using file path patterns:
   - `src/components/**`, `*.css`, `*.scss`, `*.tsx` with JSX → `ui-component`
   - `src/api/**`, `routes/**`, `endpoints/**`, `backend/**/*.py` → `api-endpoint`
   - Frontend + backend files in same task → `integration`
   - `migrations/**`, `models/**`, `schema/**`, `alembic/**` → `database`
   - `docker*`, `.github/**`, `*.yml` config, `Dockerfile*` → `infrastructure`
   - "refactor" in task description → `refactoring`
3. Multiple profiles can apply — take the union of all signals
4. If no profile matches, default to `integration` (safest)

## Plan Task Tagging Format

Every task in the plan must include:

```markdown
### Task N: [Component Name]

**Verification Profile:** ui-component
**Signals:** containers, lint, e2e (Figma-Compliance)
```

The `Signals` line is the expanded version of the profile — it tells the executing agent exactly what to run without needing to look up the profile.

## Signal Descriptions

| Signal | What It Runs | Cost |
|---|---|---|
| `containers` | `docker compose ps` — verify all services healthy | ~5s |
| `lint` | Static analysis: linters, type checkers, security scanners | ~5s |
| `property` | Property-based tests (pytest -m property / vitest property) | ~30s |
| `contract` | FE/BE API shape agreement check | ~10s |
| `mutation` | Mutation testing (mutmut / Stryker) | ~2min |
| `dst` | Fault injection via dst-simulation skill | ~3min |
| `e2e` | Playwright tests against Docker containers | ~1-3min |

Signals are always run in cost order (cheapest first). If a cheaper signal fails, skip expensive ones.
```

- [ ] **Step 2: Verify the file was created correctly**

Run: `cat .claude/skills/verification-profiles/SKILL.md | head -5`
Expected: YAML frontmatter with `name: verification-profiles`

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/verification-profiles/SKILL.md
git commit -m "feat: add verification-profiles skill — task-type to signal mapping"
```

---

## Task 2: Modify `verification-gate` — Add Per-Task Mode + Signal 6

**Files:**
- Modify: `.claude/skills/verification-gate/SKILL.md`

- [ ] **Step 1: Add per-task mode section after the pipeline overview (after line ~29)**

Find the section ending with the 5-signal pipeline table. After it, add:

```markdown
## Per-Task Mode

When invoked per-task (by `docker-verified-execution`), verification-gate accepts a **profile** — a subset of signals to run. This avoids running expensive signals on tasks that don't need them.

**Invocation:**
- Full pipeline (default): run all 5+1 signals in order
- Per-task mode: receive `signals: [containers, lint, e2e]` from the task's verification profile
- Always run `containers` first (health check gate) regardless of profile
- Skip signals not in the profile
- Cost ordering preserved within the subset

**Profile source:** Each task in the implementation plan is tagged with a verification profile (see `verification-profiles` skill). The profile expands to a signal list.

**Convergence tracking:** Scoped per-task in per-task mode. Error signatures reset between tasks.

**Assume Docker running:** In per-task mode, Docker containers are already running (set up in Task 0 by `docker-patterns`). Do NOT boot Docker — just verify containers are healthy via `docker compose ps`.
```

- [ ] **Step 2: Add Signal 6 section after Signal 5 (after line ~190)**

Find the `## Signal 5: DST Simulation` section. After it, add:

```markdown
## Signal 6: E2E Tests (~1-3 min)

Run Playwright end-to-end tests against Docker containers.

**When to run:** Signal `e2e` is in the task's profile.

**Prerequisites:** All containers healthy (Signal `containers` passed).

**Execution:**
```bash
# Per-task: run specific Playwright project based on profile
npx playwright test --project=Figma-Compliance    # for ui-component profile
npx playwright test --project=Business-Workflows  # for integration profile
npx playwright test --project=Smoke               # for quick verification

# Final verification: run all projects
npx playwright test
```

**Configuration:**
- `baseURL`: `http://localhost:<PORT>` (Docker-exposed port)
- Animations disabled for visual regression stability
- Screenshots on failure saved to `test-results/`

**Output format:**
```json
{
  "signal": "e2e",
  "status": "fail",
  "duration_seconds": 45,
  "project": "Figma-Compliance",
  "summary": { "passed": 8, "failed": 2, "skipped": 0 },
  "failures": [
    {
      "test": "homepage.figma.spec.ts > nav background color matches Figma token",
      "expected": "rgb(28, 27, 27)",
      "actual": "rgb(255, 255, 255)",
      "screenshot_diff": "test-results/nav-bg-diff.png"
    }
  ]
}
```

**stop_on_fail:** true — e2e failures indicate visible user-facing bugs.

**Fix routing:** E2e failures route to the generating agent with the failure details and screenshot diffs. The agent reads structured logs (`docker compose logs`) for backend errors that may cause frontend failures.
```

- [ ] **Step 3: Update the pipeline overview table to include Signal 6**

Find the pipeline overview table (around line 13-29). Add a row after Signal 5:

```markdown
| 6 | E2E Tests         | ~1-3 min | Playwright e2e against Docker containers    | true          |
```

- [ ] **Step 4: Update the unified output format to include Signal 6**

In the verification-report.json example (around line 193-295), add `e2e` to the signals array example alongside the existing 5.

- [ ] **Step 5: Verify changes**

Run: `grep -c "Signal 6\|e2e\|Per-Task Mode" .claude/skills/verification-gate/SKILL.md`
Expected: at least 5 matches

- [ ] **Step 6: Commit**

```bash
git add .claude/skills/verification-gate/SKILL.md
git commit -m "feat: verification-gate — add per-task mode with profiles + Signal 6 (E2E)"
```

---

## Task 3: Modify `dst-simulation` — Add Reuse Running Containers Mode

**Files:**
- Modify: `.claude/skills/dst-simulation/SKILL.md`

- [ ] **Step 1: Add reuse mode section after Phase 0 (after line ~64)**

Find `## Phase 0: Prerequisites`. After it, add:

```markdown
## Container Reuse Mode

When Docker containers are already running (set up by Task 0 in the plan), skip Phase 2-3 and go straight to fault injection.

**Detection:**
```bash
# Check if services are already running and healthy
docker compose ps --format json | jq '.[].Health'
```

**If all services healthy:**
1. Skip Phase 2 (generate compose) — use existing docker-compose.yml
2. Skip Phase 3 (boot) — containers already running
3. Add Toxiproxy as overlay:
   ```bash
   # Start only the DST overlay services alongside existing ones
   docker compose -f docker-compose.yml -f docker-compose.dst.yml up -d toxiproxy wiremock
   ```
4. Proceed to Phase 4 (fault injection)

**If services not running:**
- Fall back to full Phase 0-3 workflow (existing behavior)

**Cleanup in reuse mode:**
- Only stop Toxiproxy/WireMock overlay services
- Do NOT stop the application services (they belong to the dev environment)
```

- [ ] **Step 2: Verify changes**

Run: `grep -c "Reuse\|reuse\|overlay" .claude/skills/dst-simulation/SKILL.md`
Expected: at least 4 matches

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/dst-simulation/SKILL.md
git commit -m "feat: dst-simulation — add container reuse mode for per-task execution"
```

---

## Task 4: Modify `docker-patterns` — Add Task 0 Environment Setup

**Files:**
- Modify: `.claude/skills/docker-patterns/SKILL.md`

- [ ] **Step 1: Add Task 0 section at the end, before Anti-Patterns (before line ~344)**

Find the `## Anti-Patterns` section. Before it, add:

```markdown
## Task 0: Standing Up the Environment

When a plan includes Docker-verified execution, Task 0 sets up the complete environment before any feature work begins.

### Step 1: Generate docker-compose.yml

Scan the codebase to detect services:
- Frontend: look for `package.json` with `react`/`vue`/`next`, Vite config, etc.
- Backend: look for `requirements.txt`/`pyproject.toml` (Python), `go.mod` (Go), `package.json` with `express`/`fastapi`
- Database: look for ORM config, migration files, connection strings
- Cache: look for Redis/Memcached references

For each service:
- Create a Dockerfile (multi-stage, per patterns above)
- Add to docker-compose.yml with health check
- Configure networking (services talk by name)
- Set up volumes (bind mounts for source, named volumes for data)
- Include structured logging env vars: `LOG_FORMAT=json`, `LOG_LEVEL=info`, `SERVICE_NAME=<name>`

### Step 2: Build and Start

```bash
# Build all services
docker compose build

# Start in detached mode
docker compose up -d

# Wait for all services to be healthy
# Poll until all show "healthy" (timeout: 120s)
timeout=120; elapsed=0; interval=5
while [ $elapsed -lt $timeout ]; do
  unhealthy=$(docker compose ps --format json | jq -r '.[] | select(.Health != "healthy") | .Service')
  if [ -z "$unhealthy" ]; then
    echo "All services healthy after ${elapsed}s"
    break
  fi
  echo "Waiting for: $unhealthy"
  sleep $interval
  elapsed=$((elapsed + interval))
done
```

### Step 3: Verify Baseline

```bash
# Check all containers are running and healthy
docker compose ps

# Hit health endpoints
curl -f http://localhost:3000/health   # frontend
curl -f http://localhost:8000/health   # backend

# Verify structured logging works
docker compose logs --no-log-prefix --tail 5 backend \
  | jq -R 'fromjson? // empty | {level, service, message}'

# Run smoke tests (if they exist)
npx playwright test --project=Smoke 2>/dev/null || echo "No smoke tests yet"
```

### Step 4: Enable Watch Mode (Optional)

For faster iteration during development:

```bash
# Start with file sync (no rebuild on source changes)
docker compose watch
```

Or use volume mounts in docker-compose.override.yml:
```yaml
services:
  frontend:
    volumes:
      - ./frontend/src:/app/src
      - /app/node_modules
  backend:
    volumes:
      - ./backend:/app
      - /app/__pycache__
```

### Exit Condition

Task 0 is complete when:
- [ ] All containers show "healthy" in `docker compose ps`
- [ ] Health endpoints respond with 200
- [ ] Structured JSON logs are visible in `docker compose logs`
- [ ] Smoke tests pass (if they exist)
```

- [ ] **Step 2: Verify changes**

Run: `grep -c "Task 0\|Standing Up\|Exit Condition" .claude/skills/docker-patterns/SKILL.md`
Expected: at least 3 matches

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/docker-patterns/SKILL.md
git commit -m "feat: docker-patterns — add Task 0 environment setup workflow"
```

---

## Task 5: Create `figma-mcp-guide` Skill

**Files:**
- Create: `.claude/skills/figma-mcp-guide/SKILL.md`

- [ ] **Step 1: Create the skill file**

```markdown
---
name: figma-mcp-guide
description: "Use when a project has Figma designs or when setting up Figma MCP for the first time — guides design token extraction, component hierarchy mapping, and visual baseline generation via Figma MCP tools"
---

# Figma MCP Guide

Extract design data from Figma for testing and implementation using cursor-talk-to-figma-mcp.

## When to Use

Invoked during brainstorming when the user confirms they have Figma designs (or wants to set up Figma for a greenfield project).

## Step 0: Check Setup

Is Figma MCP configured?

```bash
# Look for Figma MCP in .mcp.json
cat .mcp.json 2>/dev/null | jq '.mcpServers["TalkToFigma"]'
```

**If not configured**, guide the user:

1. Install the Figma plugin: search "Cursor Talk To Figma MCP" in Figma Community
2. Add to `.mcp.json`:
   ```json
   {
     "mcpServers": {
       "TalkToFigma": {
         "command": "npx",
         "args": ["-y", "cursor-talk-to-figma-mcp@latest"]
       }
     }
   }
   ```
3. Open Figma, run the plugin, note the channel name
4. Verify: `join_channel` with the channel name

**If already configured**, proceed to Step 1.

## Step 1: Connect

```
join_channel({ channel: "<channel-name-from-figma-plugin>" })
get_document_info()  → understand file structure, pages, top-level frames
```

Ask the user which page/frame contains the designs to extract.

## Step 2: Extract Design Tokens

```
get_styles()
→ Returns: {
    colors: [{ id, name, key, paint }],
    texts:  [{ id, name, key, fontSize, fontName }],
    effects: [{ id, name, key }],
    grids:  [{ id, name, key }]
  }
```

Transform into `design-tokens.json`:
```json
{
  "colors": {
    "primary": "#3b82f6",
    "secondary": "#6b7280",
    "error": "#ef4444",
    "surface": "#ffffff",
    "nav-bg": "#1c1b1b",
    "ticker-accent": "#972500"
  },
  "typography": {
    "heading1": { "fontFamily": "Inter", "fontSize": "32px", "fontWeight": "700", "lineHeight": "40px" },
    "heading2": { "fontFamily": "Inter", "fontSize": "24px", "fontWeight": "600", "lineHeight": "32px" },
    "body": { "fontFamily": "Inter", "fontSize": "16px", "fontWeight": "400", "lineHeight": "24px" },
    "caption": { "fontFamily": "Inter", "fontSize": "12px", "fontWeight": "400", "lineHeight": "16px" }
  },
  "spacing": { "xs": "4px", "sm": "8px", "md": "16px", "lg": "24px", "xl": "32px" },
  "radii": { "sm": "4px", "md": "8px", "lg": "16px", "full": "9999px" }
}
```

Save to: `e2e/fixtures/design-tokens.json`

## Step 3: Map Component Hierarchy

```
scan_nodes_by_types({
  nodeId: "<page-or-frame-id>",
  types: ["FRAME", "COMPONENT", "INSTANCE"]
})
→ Returns: [{ id, name, type, bbox: { x, y, width, height } }]
```

For each key component, get detailed properties:
```
get_node_info({ nodeId: "<component-id>" })
→ Returns: {
    id, name, type,
    fills: [{ type, color (hex), opacity }],
    fontFamily, fontSize, fontWeight, lineHeightPx,
    cornerRadius,
    absoluteBoundingBox: { x, y, width, height }
  }
```

Transform into `component-map.json`:
```json
{
  "Header": {
    "selector": "header",
    "children": ["Logo", "Nav", "Search"],
    "tokens": {
      "background-color": "#1c1b1b",
      "height": "64px"
    }
  },
  "ArticleCard": {
    "selector": ".article-card",
    "children": ["Image", "Title", "Meta", "Tags"],
    "tokens": {
      "border-radius": "8px",
      "padding": "16px"
    }
  }
}
```

Save to: `e2e/fixtures/component-map.json`

## Step 4: Generate Visual Baselines (Optional)

```
export_node_as_image({
  nodeId: "<screen-id>",
  format: "png",
  scale: 1
})
→ Returns: base64-encoded PNG
```

Save to: `e2e/__screenshots__/<screen-name>.png`

Use these as Playwright `toHaveScreenshot()` baselines. Note: Figma renders may differ slightly from browser renders — update baselines after first Playwright run.

## Step 5: Mapping Rules

Use this table when writing Playwright test assertions:

| Figma Property | CSS Property | Playwright Assertion |
|---|---|---|
| `fills[0].color` | `background-color` | `getComputedStyle(el).backgroundColor` |
| `fontSize` | `font-size` | `getComputedStyle(el).fontSize` |
| `fontFamily` | `font-family` | `getComputedStyle(el).fontFamily` |
| `fontWeight` | `font-weight` | `getComputedStyle(el).fontWeight` |
| `lineHeightPx` | `line-height` | `getComputedStyle(el).lineHeight` |
| `cornerRadius` | `border-radius` | `getComputedStyle(el).borderRadius` |
| `itemSpacing` | `gap` | `getComputedStyle(el).gap` |
| `padding` | `padding` | `getComputedStyle(el).padding` |

**Color format conversion:** Figma returns hex (`#3b82f6`). `getComputedStyle()` returns rgb (`rgb(59, 130, 246)`). Convert hex to rgb for assertions or use a helper:

```typescript
function hexToRgb(hex: string): string {
  const r = parseInt(hex.slice(1, 3), 16);
  const g = parseInt(hex.slice(3, 5), 16);
  const b = parseInt(hex.slice(5, 7), 16);
  return `rgb(${r}, ${g}, ${b})`;
}
```

## MCP Tools Reference

| Tool | Purpose | Use When |
|---|---|---|
| `join_channel` | Connect to Figma plugin | Always first |
| `get_document_info` | File structure overview | Understand pages/frames |
| `get_styles` | Color, text, effect, grid styles | Extract design tokens |
| `scan_nodes_by_types` | Find components by type recursively | Map component hierarchy |
| `get_node_info` | Detailed node properties (fills, fonts, spacing) | Get exact values per component |
| `scan_text_nodes` | All text with font properties | Typography audit |
| `export_node_as_image` | PNG/SVG export | Visual regression baselines |
| `get_selection` | Currently selected nodes | Interactive exploration |

## Worked Example: auto-blog-verify-gate

The auto-blog-verify-gate project uses this pattern in `e2e/tests/homepage.spec.ts`:
- Nav background: `#1c1b1b` (dark) → asserted via `getComputedStyle().backgroundColor`
- Ticker accent: `#972500` (rust) → asserted via `getComputedStyle().color`
- Design tokens extracted from Figma, hardcoded in test file
- With this skill, tokens would instead come from `e2e/fixtures/design-tokens.json`
```

- [ ] **Step 2: Verify the file was created correctly**

Run: `head -5 .claude/skills/figma-mcp-guide/SKILL.md`
Expected: YAML frontmatter with `name: figma-mcp-guide`

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/figma-mcp-guide/SKILL.md
git commit -m "feat: add figma-mcp-guide skill — Figma MCP usage, token extraction, component mapping"
```

---

## Task 6: Create `docker-verified-execution` Skill

**Files:**
- Create: `.claude/skills/docker-verified-execution/SKILL.md`

- [ ] **Step 1: Create the skill file**

```markdown
---
name: docker-verified-execution
description: "Use when executing implementation plan tasks that must be verified against running Docker containers — orchestrates the deploy-test-diagnose-fix loop (Ralph Loop) per task with structured log diagnosis and convergence detection"
---

# Docker-Verified Execution

The deploy → test → diagnose → fix loop for each task. Every code change is verified against running Docker containers before committing.

## When to Use

During plan execution (subagent-driven-development or executing-plans). Wraps each task with Docker verification.

**Prerequisites:**
- Docker environment running (Task 0 completed via docker-patterns)
- Task tagged with a verification profile (via verification-profiles skill)
- Structured logging configured (via structured-logs skill)

## The Ralph Loop

Never let the agent judge its own fix. Docker and the test runner are the arbiters.

```
┌──────────────────────────┐
│  1. WRITE CODE           │
│     (TDD skill)          │
│     - Write failing test │
│     - Implement          │
│     - Local tests pass   │
└───────────┬──────────────┘
            ▼
┌──────────────────────────┐
│  2. DEPLOY TO DOCKER     │
│                          │
│  Option A (fast):        │
│    docker compose watch  │
│    → file sync, no build │
│                          │
│  Option B (rebuild):     │
│    docker compose up     │
│      --build <svc> -d    │
│                          │
│  Wait healthy:           │
│    poll docker compose ps│
│    timeout: 120s         │
└───────────┬──────────────┘
            ▼
┌──────────────────────────┐
│  3. RUN VERIFICATION     │
│     verification-gate    │
│     with task's profile  │
│                          │
│  e.g., signals:          │
│  [containers, lint, e2e] │
└───────────┬──────────────┘
            ▼
┌──────────────────────────┐
│  4. ALL PASS?            │
│  ├─ YES → commit,       │
│  │        next task      │
│  └─ NO  → step 5        │
└───────────┬──────────────┘
            ▼
┌──────────────────────────┐
│  5. DIAGNOSE             │
│                          │
│  Read structured logs:   │
│  docker compose logs     │
│    --no-log-prefix       │
│    --tail 100 <service>  │
│    | jq 'fromjson?       │
│      // empty            │
│      | select(.level ==  │
│        "error")'         │
│                          │
│  Categorize errors:      │
│    error_category field  │
│    → fix strategy        │
│                          │
│  Invoke systematic-      │
│  debugging with evidence │
└───────────┬──────────────┘
            ▼
┌──────────────────────────┐
│  6. LOOP GUARD           │
│                          │
│  Hash current errors:    │
│    sort error messages   │
│    → md5 hash            │
│                          │
│  Same hash as last try?  │
│  → STUCK, escalate       │
│                          │
│  Attempt >= 5?           │
│  → EXHAUSTED, escalate   │
│                          │
│  Different errors?       │
│  → PROGRESS, loop to 1  │
└──────────────────────────┘
```

## Exit Conditions

| Condition | Action |
|---|---|
| All verification signals pass | Commit changes, proceed to next task |
| Same errors 2 consecutive attempts | STUCK — escalate to user with error details and log excerpts |
| 5 attempts exhausted | EXHAUSTED — escalate to user with full history of attempts |
| Container won't start | INFRASTRUCTURE — escalate immediately, don't retry code changes |

## Log Querying Patterns

```bash
# All errors from a service
docker compose logs --no-log-prefix --tail 100 backend \
  | jq -R 'fromjson? // empty | select(.level == "error")'

# Filter by trace_id (follow a single request)
docker compose logs --no-log-prefix backend \
  | jq -R 'fromjson? // empty | select(.trace_id == "abc123")'

# Count errors by category (find dominant failure)
docker compose logs --no-log-prefix backend \
  | jq -R 'fromjson? // empty | select(.level == "error") | .error_category' \
  | sort | uniq -c | sort -rn

# Errors from last 5 minutes only
docker compose logs --no-log-prefix --since 5m backend \
  | jq -R 'fromjson? // empty | select(.level == "error")'

# Cross-service trace (follow request across services)
TRACE_ID="abc123"
docker compose logs --no-log-prefix \
  | jq -R 'fromjson? // empty | select(.trace_id == "'$TRACE_ID'")' \
  | jq -s 'sort_by(.timestamp)'
```

## Error Category → Fix Strategy

Uses the `error_category` field from structured-logs:

| Category | Meaning | Fix Strategy |
|---|---|---|
| `timeout` | Request exceeded time limit | Check query performance, add indices, increase timeout |
| `connection` | Failed upstream connection | Verify service networking, check Docker compose links |
| `validation` | Input/output schema mismatch | Fix request/response shapes, update types |
| `logic` | Business rule violation | Fix implementation logic |
| `auth` | Auth/authz failure | Fix token handling, check credentials config |
| `rate_limit` | Rate limit exceeded | Add backoff, check test parallelism |
| `unknown` | Unhandled exception | Read stack trace, add error handling |

## Deploy Strategy Selection

| Scenario | Strategy | Command |
|---|---|---|
| Source file changed (hot reload) | Watch/sync | `docker compose watch` (already running) |
| Dependency changed (package.json, requirements.txt) | Rebuild service | `docker compose up --build <service> -d` |
| Docker config changed (Dockerfile, compose) | Full rebuild | `docker compose up --build -d` |
| Database schema changed | Rebuild + migrate | `docker compose up --build -d && docker compose exec backend alembic upgrade head` |

## Health Check Polling

```bash
wait_healthy() {
  local timeout=120 elapsed=0 interval=5
  while [ $elapsed -lt $timeout ]; do
    unhealthy=$(docker compose ps --format json \
      | jq -r '.[] | select(.Health != "healthy") | .Service' 2>/dev/null)
    if [ -z "$unhealthy" ]; then
      echo "All services healthy after ${elapsed}s"
      return 0
    fi
    echo "Waiting for: $unhealthy (${elapsed}s/${timeout}s)"
    sleep $interval
    elapsed=$((elapsed + interval))
  done
  echo "TIMEOUT: services still unhealthy after ${timeout}s"
  docker compose logs --tail 20
  return 1
}
```

## Integration with Other Skills

| Skill | Role in the Loop |
|---|---|
| `test-driven-development` | Step 1 — write code with TDD |
| `docker-patterns` | Step 2 — deploy strategy, health checks |
| `verification-gate` | Step 3 — run task's verification signals |
| `structured-logs` | Step 5 — parse container logs for diagnosis |
| `systematic-debugging` | Step 5 — root cause investigation from log evidence |
| `verification-profiles` | Provides signal list per task |
```

- [ ] **Step 2: Verify the file was created correctly**

Run: `head -5 .claude/skills/docker-verified-execution/SKILL.md`
Expected: YAML frontmatter with `name: docker-verified-execution`

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/docker-verified-execution/SKILL.md
git commit -m "feat: add docker-verified-execution skill — Ralph Loop with structured log diagnosis"
```

---

## Task 7: Create `e2e-verification` Skill

**Files:**
- Create: `.claude/skills/e2e-verification/SKILL.md`

- [ ] **Step 1: Create the skill file**

```markdown
---
name: e2e-verification
description: "Use as the final verification gate after all implementation tasks are complete — runs full Playwright e2e suite against Docker containers covering Figma design compliance, business workflow validation, and structural DOM matching"
---

# E2E Verification

Final quality gate. Runs the complete Playwright e2e suite against Docker containers after all tasks are done, before finishing the development branch.

## When to Use

After all implementation tasks pass their per-task verification. This is the last gate before `finishing-a-development-branch`.

## Prerequisites

- All Docker containers healthy
- Per-task verification passed for all tasks
- `design-tokens.json` and `component-map.json` exist (if Figma was used)
- Playwright installed and configured

## Three Playwright Projects

### 1. Figma-Compliance (`testMatch: /.*figma\.spec\.ts/`)

Verifies the rendered UI matches Figma design data.

**A. Design Token Assertions**

Compare `getComputedStyle()` values against `design-tokens.json`:

```typescript
import { test, expect } from '@playwright/test';
import tokens from '../fixtures/design-tokens.json';

function hexToRgb(hex: string): string {
  const r = parseInt(hex.slice(1, 3), 16);
  const g = parseInt(hex.slice(3, 5), 16);
  const b = parseInt(hex.slice(5, 7), 16);
  return `rgb(${r}, ${g}, ${b})`;
}

async function getStyles(
  locator: import('@playwright/test').Locator,
  properties: string[]
): Promise<Record<string, string>> {
  return locator.evaluate((el, props) => {
    const cs = getComputedStyle(el);
    return Object.fromEntries(props.map(p => [p, cs.getPropertyValue(p)]));
  }, properties);
}

test('nav uses correct design tokens', async ({ page }) => {
  await page.goto('/');
  const nav = page.locator('nav');
  const styles = await getStyles(nav, ['background-color', 'height']);
  expect(styles['background-color']).toBe(hexToRgb(tokens.colors['nav-bg']));
});

test('heading typography matches Figma', async ({ page }) => {
  await page.goto('/');
  const h1 = page.locator('h1').first();
  const styles = await getStyles(h1, ['font-family', 'font-size', 'font-weight', 'line-height']);
  expect(styles['font-size']).toBe(tokens.typography.heading1.fontSize);
  expect(styles['font-weight']).toBe(tokens.typography.heading1.fontWeight);
});
```

**B. Visual Regression**

Compare screenshots against baselines:

```typescript
test('homepage visual regression', async ({ page }) => {
  await page.goto('/');
  await expect(page).toHaveScreenshot('homepage.png', {
    fullPage: true,
    maxDiffPixelRatio: 0.01,
    threshold: 0.2,
    animations: 'disabled',
    mask: [
      page.locator('.timestamp'),
      page.locator('.dynamic-content'),
    ],
  });
});

test('article card visual regression', async ({ page }) => {
  await page.goto('/');
  const card = page.locator('.article-card').first();
  await expect(card).toHaveScreenshot('article-card.png', {
    animations: 'disabled',
  });
});
```

**C. Structural Matching**

Verify DOM structure matches component-map.json:

```typescript
import componentMap from '../fixtures/component-map.json';

test('header has correct structure', async ({ page }) => {
  await page.goto('/');
  const header = page.locator(componentMap.Header.selector);
  await expect(header).toBeVisible();

  // Verify expected children exist
  for (const child of componentMap.Header.children) {
    await expect(header.locator(`[data-component="${child}"], .${child.toLowerCase()}`))
      .toHaveCount(1, { timeout: 5000 });
  }
});

test('page has correct semantic landmarks', async ({ page }) => {
  await page.goto('/');
  await expect(page.locator('header')).toHaveCount(1);
  await expect(page.locator('nav')).toHaveCount(1);
  await expect(page.locator('main')).toHaveCount(1);
  await expect(page.locator('footer')).toHaveCount(1);
});
```

### 2. Business-Workflows (`testMatch: /.*workflow\.spec\.ts/`)

Verifies user journeys work end-to-end.

```typescript
test('user can browse and read articles', async ({ page }) => {
  await page.goto('/');
  // Homepage loads with articles
  await expect(page.locator('.article-card')).toHaveCount.greaterThan(0);

  // Click first article
  await page.locator('.article-card').first().click();

  // Article page loads with content
  await expect(page.locator('article')).toBeVisible();
  await expect(page.locator('article h1')).not.toBeEmpty();
});

test('404 page renders for invalid routes', async ({ page }) => {
  const response = await page.goto('/nonexistent-page');
  expect(response?.status()).toBe(404);
  await expect(page.locator('text=not found')).toBeVisible({ timeout: 5000 });
});

test('responsive layout at mobile breakpoint', async ({ page }) => {
  await page.setViewportSize({ width: 375, height: 812 });
  await page.goto('/');
  // Mobile nav should be collapsed or hamburger
  await expect(page.locator('nav')).toBeVisible();
  // Articles should stack vertically
  const cards = page.locator('.article-card');
  if (await cards.count() >= 2) {
    const box1 = await cards.nth(0).boundingBox();
    const box2 = await cards.nth(1).boundingBox();
    expect(box2!.y).toBeGreaterThan(box1!.y); // stacked, not side-by-side
  }
});
```

### 3. Smoke (`testMatch: /.*smoke\.spec\.ts/`)

Quick critical-path check. Used per-task for fast feedback.

```typescript
test('app loads successfully', async ({ page }) => {
  const response = await page.goto('/');
  expect(response?.status()).toBe(200);
});

test('API health check responds', async ({ request }) => {
  const response = await request.get('/api/health');
  expect(response.status()).toBe(200);
});

test('primary content renders', async ({ page }) => {
  await page.goto('/');
  await expect(page.locator('main')).toBeVisible();
  await expect(page.locator('h1, h2').first()).toBeVisible();
});
```

## Playwright Configuration

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './e2e/tests',
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:5173',
    screenshot: 'only-on-failure',
    trace: 'on-first-retry',
  },
  expect: {
    toHaveScreenshot: {
      maxDiffPixelRatio: 0.01,
      threshold: 0.2,
      animations: 'disabled',
    },
  },
  projects: [
    {
      name: 'Figma-Compliance',
      testMatch: /.*figma\.spec\.ts/,
    },
    {
      name: 'Business-Workflows',
      testMatch: /.*workflow\.spec\.ts/,
    },
    {
      name: 'Smoke',
      testMatch: /.*smoke\.spec\.ts/,
      retries: 0,
    },
  ],
});
```

## Output

```json
{
  "e2e_report": {
    "figma_compliance": {
      "token_tests": { "passed": 12, "failed": 0 },
      "visual_regression": { "passed": 5, "failed": 1, "diffs": ["test-results/hero-diff.png"] },
      "structural": { "passed": 8, "failed": 0 }
    },
    "business_workflows": {
      "journeys": { "passed": 6, "failed": 0 }
    },
    "smoke": {
      "tests": { "passed": 3, "failed": 0 }
    },
    "overall": "FAIL",
    "failure_summary": "1 visual regression failure in hero section"
  }
}
```

## Integration

- Runs AFTER all per-task verification passes
- Runs BEFORE `finishing-a-development-branch`
- If failures: loop back to `docker-verified-execution` for the failing area
- Reports feed into the commit/PR description
```

- [ ] **Step 2: Verify the file was created correctly**

Run: `head -5 .claude/skills/e2e-verification/SKILL.md`
Expected: YAML frontmatter with `name: e2e-verification`

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/e2e-verification/SKILL.md
git commit -m "feat: add e2e-verification skill — final Playwright gate with Figma + business workflows"
```

---

## Task 8: Modify `brainstorming` — Add Figma Question to Checklist

**Files:**
- Modify: `.claude/skills/brainstorming/SKILL.md`

- [ ] **Step 1: Add Figma question to the checklist**

Find the checklist section (around line 22-33). Replace the existing items 1-2 with:

```markdown
## Checklist

You MUST create a task for each of these items and complete them in order:

1. **Explore project context** — check files, docs, recent commits
2. **Offer visual companion** (if topic will involve visual questions) — this is its own message, not combined with a clarifying question. See the Visual Companion section below.
3. **Ask about Figma designs** — "Do you have Figma designs for this project?"
   Always ask explicitly, even for greenfield projects. If yes → invoke
   figma-mcp-guide skill (helps set up MCP if not configured, then extracts
   design tokens, component map, and visual baselines). If no → skip,
   proceed without Figma-based verification.
4. **Ask clarifying questions** — one at a time, understand purpose/constraints/success criteria
5. **Propose 2-3 approaches** — with trade-offs and your recommendation
6. **Present design** — in sections scaled to their complexity, get user approval after each section
7. **Write design doc** — save to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` and commit
8. **Spec self-review** — quick inline check for placeholders, contradictions, ambiguity, scope (see below)
9. **User reviews written spec** — ask user to review the spec file before proceeding
10. **Transition to implementation** — invoke writing-plans skill to create implementation plan
```

- [ ] **Step 2: Verify changes**

Run: `grep -c "Figma\|figma-mcp-guide" .claude/skills/brainstorming/SKILL.md`
Expected: at least 2 matches

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/brainstorming/SKILL.md
git commit -m "feat: brainstorming — add Figma design question to checklist (invokes figma-mcp-guide)"
```

---

## Self-Review

**1. Spec coverage:**
- Per-task verification mode → Task 2 (verification-gate modification)
- Configurable signals → Task 1 (verification-profiles) + Task 2 (per-task mode)
- Docker ground truth → Task 4 (docker-patterns Task 0) + Task 6 (docker-verified-execution)
- Structured log debugging → Task 6 (log querying patterns + error category mapping)
- Figma MCP extraction → Task 5 (figma-mcp-guide)
- Final e2e gate → Task 7 (e2e-verification)
- Brainstorming integration → Task 8 (brainstorming checklist edit)
- DST reuse mode → Task 3 (dst-simulation modification)
- Sub-agent loop → Task 6 (Ralph Loop with convergence detection)
- All spec requirements covered.

**2. Placeholder scan:** No TBDs, TODOs, or vague instructions. All tasks have exact file paths, exact content, and exact verification commands.

**3. Type consistency:**
- `verification-profiles` defines profile names → same names used in `docker-verified-execution` and `verification-gate`
- `design-tokens.json` path (`e2e/fixtures/`) consistent across `figma-mcp-guide` and `e2e-verification`
- `component-map.json` path consistent across `figma-mcp-guide` and `e2e-verification`
- Signal names (containers, lint, property, contract, mutation, dst, e2e) consistent across all skills
- Error category vocabulary consistent between `structured-logs` and `docker-verified-execution`
