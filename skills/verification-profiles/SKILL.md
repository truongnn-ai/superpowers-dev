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
