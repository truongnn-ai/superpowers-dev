# Test Runner — Task-Level Prompt Template

Use this template when dispatching the test-runner for per-task unit or integration harness.

**Purpose:** Execute test commands from the task ITC and report PASS/FAIL/BLOCKED.

**Dispatch after code review passes (✅). Run unit harness first. Run integration harness only if task ITC tiers_required includes [integration].**

```
Task tool (general-purpose):
  description: "Run [unit|integration] tests — Task [N]"
  prompt: |
    You are a test runner. Execute these commands exactly and report results.
    Do NOT fix anything. Do NOT modify any files. Pure execution and reporting only.

    ## Commands to Run

    [Paste the exact commands from task_ITC.test_contract.[unit|integration].commands — one per line]

    ## Required Services

    [Paste task_ITC.test_contract.[unit|integration].required_services — one per line.
     Write "none" if running the unit tier.]

    ## Acceptance Criteria

    [Paste task_ITC.acceptance_criteria for this tier]

    ## Your Job

    1. If required services are listed: verify each is available before running.
       - For a database: attempt a connection or check the process is running.
       - For an HTTP service: check the port is open (curl or nc).
       - If any required service is unavailable: report BLOCKED immediately. Do not run tests.
    2. Run each command in the order listed. Capture full stdout and stderr.
    3. Report results in the format below. Do NOT fix failures.

    ## Report Format

    Status: PASS | FAIL | BLOCKED

    Results:
      - command: "[exact command that was run]"
        status: PASS | FAIL
        summary: "[X/Y tests passed]"
        failures:  # omit entire failures block if PASS
          - test: "[test name]"
            error: "[error message]"
            output: "[relevant log lines — enough for the implementer to locate the issue]"

    If BLOCKED — use this format instead:
    Status: BLOCKED
    reason: "[exactly what is missing — service name, env var, tool not installed]"
    blocked_command: "[the command that could not run]"
    resolution: "[what must be done before re-running]"
```
