# Coding Agent — ITC Negotiation Prompt Template

Use this template when dispatching the coding agent for ITC negotiation.

**Purpose:** Propose what will be built and identify natural test seams.

**Rounds:**
- Round 1: coding-agent proposes draft ITC (no prior context from testing agent)
- Round 3 (if needed): coding-agent responds to testing-agent amendments

```
Task tool (general-purpose):
  description: "ITC Negotiation Round [1|3] - Coding Agent: Task [N]"
  prompt: |
    You are the coding agent in an Implementation-Testing Contract (ITC) negotiation.
    Your role: propose exactly what will be built and identify natural test seams.
    You are NOT implementing yet — only defining the contract.

    ## Task Specification

    [FULL TEXT of task from plan — paste here, do not reference a file]

    ## Existing Codebase Context

    [Scan the codebase before filling this in. Include:
    - Relevant existing files the implementation will touch
    - Established patterns and conventions to follow
    - Known interfaces or types this code will use
    Omit entirely if this is a new project with no existing code.]

    ## Testing Agent's Amendments (Round 3 only — omit entirely on Round 1)

    [Paste the testing agent's full Round 2 response here.
    Accept or dispute specific items with technical reasoning.
    End with ✅ if you accept the final contract.]

    ## Your Job

    Produce a draft ITC in the YAML format below. Be specific:
    - Exact file paths for everything you will create or modify
    - Every external dependency or service boundary
    - tiers_required: [unit] or [unit, integration]
    - For integration tier: list required services precisely
    - Any "cannot reasonably test X at runtime" with specific reasoning

    ## ITC Format

    ```yaml
    task_id: [N]
    task_name: "[task name]"

    implementation_scope:
      components:
        - "[component description] ([exact/file/path.ext])"
      integration_surfaces:
        - "[external boundary — what service/DB/API is crossed]"

    test_contract:
      tiers_required: [unit]  # or [unit, integration]
      rationale: "[one sentence: why these tiers]"

      unit:
        commands:
          - "[exact runnable command targeting specific test file]"
        must_cover:
          - "[specific behavior to verify]"

      # Include integration section only if tiers_required: [unit, integration]
      integration:
        commands:
          - "[exact runnable command targeting specific test file]"
        must_cover:
          - "[specific behavior to verify]"
        required_services:
          - "[service name and type — e.g., 'PostgreSQL test database on port 5432']"

    acceptance_criteria:
      unit: all pass
      integration: all pass  # omit if tiers_required: [unit] only
      no_regression: true

    sign_off:
      coding_agent: ✅
      testing_agent: (pending)
    ```

    ## If Producing a Solution ITC

    When dispatched for solution-level ITC negotiation (after all tasks complete),
    scan the actual implementation first, then use this YAML format instead:

    ```yaml
    solution: "[feature or project name]"
    tasks_covered: [1, 2, 3]  # list all completed task IDs

    implementation_summary:
      entry_points:
        - "[HTTP method] [route or component path]"
      protected_routes:  # omit if not applicable
        - "[HTTP method] [route]"

    test_contract:
      tiers_required: [e2e, full_suite]
      rationale: "[one sentence: all tasks complete, verify full user journey end-to-end]"

      e2e:
        commands:
          - "[exact E2E command — e.g., npx playwright test tests/e2e/feature.spec.ts]"
        scenarios:
          - "[user flow to verify — e.g., 'user login and session persistence']"
        required_services:
          - "[what must be running — e.g., 'backend API on port 3001', 'database seeded', 'frontend built']"

      full_suite:
        command: "[exact command to run full test suite — e.g., npm test]"
        rationale: "Catch any cross-task regression not visible in individual task ITCs"

    acceptance_criteria:
      e2e: all scenarios pass
      full_suite: all pass
      no_regression: true

    sign_off:
      coding_agent: ✅
      testing_agent: (pending)
    ```

    End your response with ✅ to confirm you have produced your proposal.
```
