# Docker-Verified Figma E2E Pipeline — Design Spec

## Goal

Extend the superpowers skill pipeline with Docker-first verified execution, Figma MCP design extraction, and comprehensive Playwright e2e testing — so that every sub-agent task is verified against running containers and the final output matches both Figma designs and business requirements.

## Architecture

Layered extension approach: superpowers skills stay untouched (except 1 line in brainstorming checklist). Custom skills augment the pipeline at specific hook points. Existing custom skills (`verification-gate`, `dst-simulation`, `docker-patterns`, `structured-logs`) receive targeted modifications. Four new skills fill the gaps.

## Core Principles

1. **Docker is ground truth** — all verification runs against containers, not local dev server
2. **Verification per task** — catch runtime/integration errors early, not just at the end
3. **Structured logs are the debugging source** — when containers fail, parse JSON logs to diagnose
4. **Configurable signals per task** — UI tasks don't need mutation testing; API tasks don't need visual regression
5. **Sub-agents loop until Docker-verified** — deploy → test → read logs → fix → retry

---

## End-to-End Flow

```
USER: "Add a news ticker component to the homepage"
│
▼
┌─────────────────────────────────────────────────────────┐
│  1. BOOTSTRAP (unchanged)                               │
│  SessionStart → using-superpowers injected               │
└─────────────────────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────┐
│  2. BRAINSTORMING (1 line added to checklist)            │
│                                                         │
│  Explore context → clarifying questions                 │
│      │                                                  │
│      ├─→ Ask: "Do you have Figma designs for this       │
│      │   project?" (always ask, even if .mcp.json       │
│      │   exists — greenfield projects may want to       │
│      │   set up Figma MCP from scratch)                 │
│      │   ├─ YES → invoke figma-mcp-guide                │
│      │   │   • Help set up Figma MCP if not configured  │
│      │   │   • Connect via join_channel                 │
│      │   │   • Extract tokens via get_styles            │
│      │   │   • Map component hierarchy                  │
│      │   │   • Output: design-tokens.json + component   │
│      │   │     map                                      │
│      │   └─ NO → skip, proceed without Figma tokens     │
│      │                                                  │
│  Propose approaches → present design → write spec       │
│  Spec includes: Figma tokens + verification requirements│
└─────────────────────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────┐
│  3. WRITING-PLANS (unchanged, plan content richer)      │
│                                                         │
│  Task 0: Docker environment setup                       │
│      • docker-compose.yml with all services             │
│      • Health checks, networking, volumes               │
│      • Verify all containers healthy                    │
│      • verification: [containers]                       │
│                                                         │
│  Task 1-N: Feature tasks, each tagged with a            │
│  verification profile:                                  │
│      • ui-component:  [containers, lint, e2e]           │
│      • api-endpoint:  [containers, lint, property,      │
│                        contract]                        │
│      • integration:   [containers, lint, e2e, contract] │
│      • database:      [containers, lint, property,      │
│                        contract]                        │
│      • infrastructure:[containers, lint]                │
│      • refactoring:   [containers, lint, property, e2e, │
│                        contract]                        │
│                                                         │
│  verification-profiles skill informs these tags         │
└─────────────────────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────┐
│  4. EXECUTION (docker-verified loop per sub-agent)      │
│                                                         │
│  subagent-driven-development dispatches per task:       │
│                                                         │
│  ┌───────────────────────────────────────────────┐      │
│  │ docker-verified-execution (per sub-agent)      │      │
│  │                                                │      │
│  │  1. Write code (TDD skill)                     │      │
│  │  2. Deploy to Docker                           │      │
│  │     - compose watch (fast) or rebuild service  │      │
│  │     - wait for all services healthy            │      │
│  │  3. Run task's verification signals            │      │
│  │     (verification-gate with task's profile)    │      │
│  │  4. All pass? → commit + next task             │      │
│  │     Fail? → continue to step 5                 │      │
│  │  5. Diagnose                                   │      │
│  │     - docker compose logs | jq (structured)    │      │
│  │     - Categorize via error_category            │      │
│  │     - Invoke systematic-debugging              │      │
│  │  6. Loop guard                                 │      │
│  │     - Hash current errors                      │      │
│  │     - Same hash 2x → STUCK, escalate           │      │
│  │     - Attempt >= 5 → EXHAUSTED, escalate       │      │
│  │     - Different errors → PROGRESS, loop to 1   │      │
│  └───────────────────────────────────────────────┘      │
│                                                         │
│  Code review between tasks (requesting-code-review)     │
└─────────────────────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────┐
│  5. FINAL VERIFICATION                                  │
│                                                         │
│  e2e-verification skill runs full Playwright suite:     │
│                                                         │
│  A. Figma compliance (--project=Figma-Compliance)       │
│     • Design token assertions (getComputedStyle vs      │
│       design-tokens.json)                               │
│     • Visual regression (toHaveScreenshot vs baselines)  │
│     • Structural matching (DOM hierarchy vs component   │
│       map)                                              │
│                                                         │
│  B. Business workflows (--project=Business-Workflows)   │
│     • User journeys (browse → read → interact)          │
│     • Error flows (404, empty states, edge cases)       │
│     • Cross-service integration                         │
│                                                         │
│  All run against Docker containers                      │
│  → e2e-report.json                                      │
└─────────────────────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────┐
│  6. FINISH (unchanged)                                  │
│  finishing-a-development-branch                         │
│  → Merge / PR / Keep / Discard                          │
└─────────────────────────────────────────────────────────┘
```

---

## Existing Skills — Required Modifications

### verification-gate

**Add per-task mode with configurable signal selection:**

- Accept a `profile` parameter: list of signal names to run
- `containers` signal always runs first (health check gate)
- Skip signals not in the profile
- Convergence tracking scoped per-task

**Add Signal 6: E2E Tests:**

- Runs Playwright against Docker containers (`baseURL: http://localhost:PORT`)
- Supports project selection: `--project=Figma-Compliance`, `--project=Business-Workflows`, `--project=Smoke`
- Output: test results + screenshot diffs (on failure)
- Cost: ~1-3min (runs after cheaper signals)
- Position: after Signal 5 (DST) in cost ordering

**Assume Docker already running:**

- Docker environment set up in Task 0
- verification-gate runs tests against existing containers
- No self-booting of Docker environment

### dst-simulation

**Add "reuse running containers" mode:**

- Phase 0 check: if containers already running (`docker compose ps`), skip Phase 2-3
- Go straight to Phase 4 (fault injection)
- Overlay mode: add Toxiproxy sidecar to existing compose rather than generating a new one

### docker-patterns

**Add "Task 0: Environment Setup" workflow section:**

- Step 1: Generate docker-compose.yml — detect services from codebase, apply patterns, include health checks
- Step 2: Build and start — `docker compose up --build -d`, wait for all healthy
- Step 3: Verify baseline — hit health endpoints, run smoke tests, confirm structured logging works
- Step 4: Enable watch mode — `docker compose watch` for file sync without rebuild
- Exit condition: all containers healthy + smoke tests pass

### structured-logs

No modifications needed. Log querying patterns covered by `docker-verified-execution` skill.

---

## New Skills

### figma-mcp-guide

**Triggers during:** Brainstorming (when project has Figma MCP configured)

**Purpose:** Teach the agent how to use Figma MCP tools to extract design data.

**Workflow:**

0. **Check setup** — Is Figma MCP configured? Look for `.mcp.json` with `cursor-talk-to-figma-mcp` or Figma MCP tools available in the environment. If not configured, guide the user through setup: install the Figma plugin, configure `.mcp.json`, verify connection. For greenfield projects, this is the entry point.
1. **Connect** — `join_channel` to Figma plugin, `get_document_info` for file structure
2. **Extract design tokens** — `get_styles` for catalog of colors, text styles, effects, grids. Output as `design-tokens.json`:
   ```json
   {
     "colors": { "primary": "#3b82f6", "error": "#ef4444" },
     "typography": { "heading1": { "fontFamily": "Inter", "fontSize": "32px", "fontWeight": "700", "lineHeight": "40px" } },
     "spacing": { "sm": "8px", "md": "16px", "lg": "24px" },
     "radii": { "sm": "4px", "md": "8px" }
   }
   ```
3. **Map component hierarchy** — `scan_nodes_by_types(types: ["FRAME", "COMPONENT", "INSTANCE"])` for tree, `get_node_info` per key component for detailed properties. Output as `component-map.json`:
   ```json
   {
     "Header": { "selector": "header", "children": ["Logo", "Nav", "Search"], "tokens": {} },
     "ArticleCard": { "selector": ".article-card", "children": ["Image", "Title", "Meta"], "tokens": {} }
   }
   ```
4. **Generate visual baselines** (optional) — `export_node_as_image(format: "png")` for key screens, save to `e2e/__screenshots__/`
5. **Mapping rules** — Figma node property → CSS property → Playwright assertion:

| Figma Property | CSS Property | Playwright |
|---|---|---|
| `fills[0].color` | `background-color` | `getComputedStyle().backgroundColor` |
| `fontSize` | `font-size` | `getComputedStyle().fontSize` |
| `fontFamily` | `font-family` | `getComputedStyle().fontFamily` |
| `fontWeight` | `font-weight` | `getComputedStyle().fontWeight` |
| `lineHeightPx` | `line-height` | `getComputedStyle().lineHeight` |
| `cornerRadius` | `border-radius` | `getComputedStyle().borderRadius` |
| `itemSpacing` | `gap` | `getComputedStyle().gap` |
| `padding` | `padding` | `getComputedStyle().padding` |

**MCP tools used:** `join_channel`, `get_styles`, `scan_nodes_by_types`, `get_node_info`, `scan_text_nodes`, `export_node_as_image`

**Reference:** auto-blog-verify-gate as worked example (existing homepage.spec.ts design token tests).

### docker-verified-execution

**Triggers during:** Step 4 (Execution), wraps each sub-agent's task

**Purpose:** The deploy → test → diagnose → fix loop (Ralph Loop pattern)

**The loop:**

1. **Write code** — TDD skill (write failing test → implement → local tests pass)
2. **Deploy to Docker** — `docker compose watch` (file sync, fast) or `docker compose up --build <service> -d` (rebuild). Wait for all services healthy via polling `docker compose ps` (timeout: 120s)
3. **Run verification** — invoke `verification-gate` with task's profile (e.g., `[containers, lint, e2e]`)
4. **All pass?** — YES → commit + next task. NO → continue to step 5
5. **Diagnose** — Read structured logs: `docker compose logs --no-log-prefix --tail 100 <service> | jq 'fromjson? // empty | select(.level == "error")'`. Categorize errors via `error_category` from logs. Invoke `systematic-debugging` with log evidence.
6. **Loop guard** — Hash current errors. Same hash as last attempt → STUCK, escalate to user. Attempt >= 5 → EXHAUSTED, escalate. Different errors → PROGRESS, loop back to step 1.

**Exit conditions:**
- All verification signals pass → commit + next task
- Same errors 2x in a row → STUCK, ask user for guidance
- 5 attempts exhausted → EXHAUSTED, ask user for guidance

**Log querying patterns:**
```bash
# All errors from a service
docker compose logs --no-log-prefix --tail 100 backend \
  | jq -R 'fromjson? // empty | select(.level == "error")'

# Filter by trace_id
docker compose logs --no-log-prefix backend \
  | jq -R 'fromjson? // empty | select(.trace_id == "abc123")'

# Count errors by category
docker compose logs --no-log-prefix backend \
  | jq -R 'fromjson? // empty | select(.level == "error") | .error_category' \
  | sort | uniq -c | sort -rn
```

### verification-profiles

**Triggers during:** Step 3 (Writing Plans) — informs task tagging

**Purpose:** Define which verification signals each task type requires

**Profile definitions:**

| Profile | Signals | Playwright Project | Pytest Marker |
|---|---|---|---|
| `ui-component` | containers, lint, e2e | Figma-Compliance | — |
| `api-endpoint` | containers, lint, property, contract | — | `api` |
| `integration` | containers, lint, e2e, contract | Business-Workflows | `integration` |
| `database` | containers, lint, property, contract | — | `db` |
| `infrastructure` | containers, lint | — | `smoke` |
| `refactoring` | containers, lint, property, e2e, contract | Business-Workflows | — (run all) |

**Profile selection rules:**

1. Parse task description + file paths in plan
2. Match against profiles:
   - `src/components/**`, `*.css`, `*.scss` → `ui-component`
   - `src/api/**`, `routes/**`, `endpoints/**` → `api-endpoint`
   - Frontend + backend files in same task → `integration`
   - `migrations/**`, `models/**`, `schema/**` → `database`
   - `docker*`, `.github/**`, `*.yml` config → `infrastructure`
   - "refactor" in task description → `refactoring`
3. Tag task in plan: `**Verification Profile:** ui-component`
4. Multiple profiles can apply (union of signals)

**Plan task format:**
```markdown
### Task 3: Add news ticker component

**Files:**
- Create: `src/components/NewsTicker.tsx`
- Create: `e2e/tests/ticker.figma.spec.ts`

**Verification Profile:** ui-component
**Signals:** containers, lint, e2e (Figma-Compliance)
```

### e2e-verification

**Triggers during:** Step 5 (Final verification, after all tasks complete)

**Purpose:** Run the complete Playwright e2e suite against Docker containers

**Three Playwright projects:**

**1. Figma-Compliance** (`testMatch: /.*figma\.spec\.ts/`)
- Design token assertions: `getComputedStyle()` vs `design-tokens.json` values (colors as rgb, fonts, spacing, radii)
- Visual regression: `toHaveScreenshot()` with `maxDiffPixelRatio: 0.01`, `threshold: 0.2`, `animations: 'disabled'`, mask dynamic content
- Structural matching: DOM hierarchy snapshot vs `component-map.json`, element counts, semantic HTML landmarks

**2. Business-Workflows** (`testMatch: /.*workflow\.spec\.ts/`)
- User journeys: complete flows (browse → read → interact), form submissions, navigation
- Cross-service integration: frontend calls backend API correctly, data displays accurately
- Edge cases: empty states, long content, special characters, responsive breakpoints

**3. Smoke** (`testMatch: /.*smoke\.spec\.ts/`)
- Critical path only: app loads, key pages render, primary action works
- Used per-task (fast subset), not just final verification

**Configuration:**
```typescript
// playwright.config.ts
export default defineConfig({
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
  },
  expect: {
    toHaveScreenshot: {
      maxDiffPixelRatio: 0.01,
      threshold: 0.2,
      animations: 'disabled',
    },
  },
  projects: [
    { name: 'Figma-Compliance', testMatch: /.*figma\.spec\.ts/ },
    { name: 'Business-Workflows', testMatch: /.*workflow\.spec\.ts/ },
    { name: 'Smoke', testMatch: /.*smoke\.spec\.ts/, retries: 0 },
  ],
});
```

**Output:** `e2e-report.json` with per-project results (passed/failed counts, failure details, screenshot diffs).

---

## Superpowers Integration Point

**Single edit to `brainstorming/SKILL.md` checklist:**

```markdown
## Checklist

1. **Explore project context** — check files, docs, recent commits
2. **Offer visual companion** (if visual questions ahead)
3. **Ask about Figma designs** — "Do you have Figma designs for this project?"
   Always ask explicitly, even for greenfield projects. If yes → invoke
   figma-mcp-guide skill (helps set up MCP if not configured, then extracts
   design tokens, component map, and visual baselines). If no → skip,
   proceed without Figma-based verification.
4. **Ask clarifying questions** — one at a time
5. **Propose 2-3 approaches** — with trade-offs and recommendation
6. **Present design** — include verification strategy per component
7. **Write design doc** — save spec and commit
8. **Spec self-review** — check for placeholders, contradictions, ambiguity
9. **User reviews written spec**
10. **Transition to implementation** — invoke writing-plans skill
```

All other superpowers skills remain unchanged.

---

## Skill Inventory Summary

| Skill | Status | Category |
|---|---|---|
| `verification-gate` | Modify (per-task mode, Signal 6, assume Docker running) | Existing |
| `dst-simulation` | Modify (reuse running containers mode) | Existing |
| `docker-patterns` | Modify (Task 0 setup workflow) | Existing |
| `structured-logs` | No change | Existing |
| `figma-mcp-guide` | New | Design extraction |
| `docker-verified-execution` | New | Execution loop |
| `verification-profiles` | New | Task configuration |
| `e2e-verification` | New | Final verification |

## Tech Stack

- **Figma:** cursor-talk-to-figma-mcp (39 tools via WebSocket relay)
- **E2E Testing:** Playwright with multi-project config
- **Containers:** Docker Compose with native health checks + `docker compose watch`
- **Log Parsing:** `docker compose logs` + jq for structured JSON
- **Convergence:** Error hash comparison + max retry guard (Ralph Loop pattern)
- **Design Tokens:** Extracted from Figma → JSON → shared by app CSS and test assertions

## Reference Implementation

auto-blog-verify-gate project serves as worked example:
- Existing Playwright e2e tests (2 suites, 9 tests)
- Existing Figma MCP config with design token validation
- Existing property-based tests (fast-check + vitest)
- Existing mutation testing (Stryker)
- Existing structured logging (structlog on backend)
