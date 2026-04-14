# Testing Agent — ITC Negotiation Prompt Template

Use this template when dispatching the testing agent for ITC negotiation.

**Purpose:** Review the coding agent's draft, challenge missing coverage, finalize test commands and acceptance criteria.

**Only dispatch after the coding agent produces a draft (Round 2, or re-review if dispute continues past Round 3).**

```
Task tool (general-purpose):
  description: "ITC Negotiation Round 2 - Testing Agent: Task [N]"
  prompt: |
    You are the testing agent in an Implementation-Testing Contract (ITC) negotiation.
    Your role: ensure the contract has rigorous, executable test coverage.
    You are NOT implementing — only reviewing and strengthening the test contract.

    ## Task Specification

    [FULL TEXT of task from plan — paste here]

    ## Coding Agent's Draft ITC

    [Full ITC YAML from the coding agent's response — paste here]

    ## Your Job

    Review the draft ITC and check ALL of the following:

    1. **tiers_required is correct**
       - Does this task cross a service boundary (DB, external API, filesystem)?
       - If yes and coding agent said [unit] only — that is wrong. Add integration.
       - unit is always the minimum tier. Never approve a contract without unit.

    2. **Test commands are exact and runnable**
       - Commands must be copy-pasteable. Not "npm test" alone — require specific file paths.
       - Every behavior in must_cover must have a command that exercises it.
       - Add commands for anything not covered.

    3. **must_cover is complete**
       - Happy path covered?
       - Error cases: invalid input, service failure, boundary conditions?
       - Every behavior stated in the task spec is reflected somewhere?

    4. **required_services is complete (integration tier)**
       - What exactly must be running? Be specific (e.g., "PostgreSQL on port 5432 with test schema seeded").
       - Are env vars required? List them.

    5. **"Untestable" declarations are valid**
       - If the coding agent declared something cannot be runtime-tested, challenge it if a test approach exists.
       - Provide the concrete test approach if one does exist.

    Sign ✅ if the contract is complete and you accept it without changes.

    If amendments are needed: list each one specifically with the exact addition —
    not "add more error cases" but the exact must_cover item or command to add.

    End your response with the complete updated ITC YAML including your sign-off:

    ```yaml
    sign_off:
      coding_agent: ✅
      testing_agent: ✅   # only if you fully approve — write ❌ with amendments if not
    ```
```
