# Changelog
## [v1.1.0] - 2026-05-11

### Added
- **Subagent-driven development tier model**: tier rubric + tier-aware dispatch in the inner loop.
- **ITC negotiation protocol**: coding-agent and testing-agent negotiation prompts, escalation handling, journey-context resolution, end-of-run ledger.
- **E2E coverage matrix**: testing-strategies playbook, coverage-matrix design spec, plan, and integration into subagent-driven-development.
- **E2E test harness**: design spec, plan, prompt templates (`coding-agent`, `testing-agent`, `test-runner-task`, `test-runner-solution`), `contracts/` directory.
- **Fullstack adoption guide** (`docs/guides/`): quick-start, daily-workflows, skills-cheatsheet, team-leads, README, SDLC mental model and workflow.

### Changed
- `subagent-driven-development` SKILL: tier dispatch, ITC integration, escalation handling.
- `writing-plans` SKILL: wires tier rubric, adds Journeys reference and `contributes_to` field.
- `brainstorming` SKILL: Journey Enumeration step.
- `implementer-prompt`, `spec-reviewer-prompt`, `testing-agent-prompt`, `coding-agent-prompt`: tier-aware exits, ITC blocks, journey context.

-----

## [5.0.5] - 2026-03-17

### Fixed

- **Brainstorm server ESM fix**: Renamed `server.js` → `server.cjs` so the brainstorming server starts correctly on Node.js 22+ where the root `package.json` `"type": "module"` caused `require()` to fail. ([PR #784](https://github.com/obra/superpowers/pull/784) by @sarbojitrana, fixes [#774](https://github.com/obra/superpowers/issues/774), [#780](https://github.com/obra/superpowers/issues/780), [#783](https://github.com/obra/superpowers/issues/783))
- **Brainstorm owner-PID on Windows**: Skip `BRAINSTORM_OWNER_PID` lifecycle monitoring on Windows/MSYS2 where the PID namespace is invisible to Node.js. Prevents the server from self-terminating after 60 seconds. The 30-minute idle timeout remains as the safety net. ([#770](https://github.com/obra/superpowers/issues/770), docs from [PR #768](https://github.com/obra/superpowers/pull/768) by @lucasyhzhu-debug)
- **stop-server.sh reliability**: Verify the server process actually died before reporting success. Waits up to 2 seconds for graceful shutdown, escalates to `SIGKILL`, and reports failure if the process survives. ([#723](https://github.com/obra/superpowers/issues/723))

### Changed

- **Execution handoff**: Restore user choice between subagent-driven-development and executing-plans after plan writing. Subagent-driven is recommended but no longer mandatory. (Reverts `5e51c3e`)
