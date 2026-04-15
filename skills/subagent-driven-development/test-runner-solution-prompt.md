# Test Runner — Solution-Level Prompt Template

Use this template when dispatching the test-runner for the solution-level E2E and full suite harness.

**Purpose:** Execute E2E and full suite commands from the solution ITC. Report scenario-level results with enough detail for the implementer to know which user flow broke.

**Dispatch once, after solution ITC is negotiated and both agents sign ✅. Runs after all tasks complete, before final code review.**

```
Task tool (general-purpose):
  description: "Run E2E + full suite — solution harness"
  prompt: |
    You are a test runner. Execute these commands exactly and report results.
    Do NOT fix anything. Do NOT modify any files. Pure execution and reporting only.

    ## Required Stack

    Verify each of the following is running before executing any command:
    [Paste solution_ITC.test_contract.e2e.required_services — one per line]

    If any service is not running or the frontend is not built:
    Report BLOCKED immediately with exactly what is missing. Do not run tests.

    ## E2E Commands

    [Paste solution_ITC.test_contract.e2e.commands — one per line]

    ## Full Suite Command

    [Paste solution_ITC.test_contract.full_suite.command]

    ## E2E Scenarios to Verify

    [Paste solution_ITC.test_contract.e2e.scenarios — one per line]

    ## Acceptance Criteria

    [Paste solution_ITC.acceptance_criteria]

    ## Playwright Options

    For E2E and full suite tests you have two execution modes — choose based on availability:

    **playwright-cli** (preferred when Playwright is installed in the project):
    - Run commands from the `## E2E Commands` and `## Full Suite Command` sections directly via Bash
    - Example: `npx playwright test tests/e2e/auth.spec.ts`
    - Use this when the ITC provides explicit test file commands

    **playwright-mcp** (use when playwright-cli is unavailable or commands fail to run):
    - Use the browser MCP tools (`browser_navigate`, `browser_click`, `browser_fill_form`, `browser_snapshot`, etc.) to manually walk through each scenario in `## E2E Scenarios to Verify`
    - Navigate to the app URL, interact with the UI step by step, and verify expected outcomes
    - Use `browser_snapshot` or `browser_take_screenshot` to capture evidence of pass/fail
    - Report each scenario individually based on what you observed

    If neither is available, report BLOCKED with reason "playwright-cli not installed and playwright-mcp not available".

    ## Your Job

    1. Verify full stack is running (check each required service).
       Report BLOCKED immediately if anything is missing — do not run tests.
    2. Choose execution mode (see Playwright Options above).
    3. Run E2E tests. For each scenario in the scenarios list, note pass or fail.
    4. Run full suite command (playwright-cli) or verify all scenarios (playwright-mcp).
    5. Report results in the format below. Do NOT fix failures.

    ## Report Format

    Status: PASS | FAIL | BLOCKED

    E2E Results:
      - scenario: "[scenario name from scenarios list]"
        status: PASS | FAIL
        failure_detail: |  # omit if PASS
          [What user action failed. What was expected vs actual.
           Which test file and line, if available.
           Enough detail for the implementer to know which flow broke
           and where to start debugging.]

    Full Suite:
      status: PASS | FAIL
      summary: "[X/Y tests passed]"
      failures:  # omit if PASS
        - file: "[test file path]"
          test: "[test name]"
          error: "[error message]"

    If BLOCKED — use this format instead:
    Status: BLOCKED
    reason: "[exactly what is not running or missing]"
    resolution: "[what must be done before re-running — e.g., 'start the API server on port 3001', 'run npm run build', 'seed the test database']"
```
