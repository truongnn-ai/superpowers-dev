# Coding Agent — ITC Negotiation Prompt Template

Use this template when dispatching the coding agent for ITC negotiation.

**Purpose:** Propose what will be built and identify natural test seams.

**Rounds:**
- Round 1: coding-agent proposes draft ITC (no prior context from testing agent)
- Round 3: coding-agent responds to testing-agent Round 2 amendments
- Round 5 (if needed): coding-agent final response to testing-agent Round 4 amendments

```
Task tool (general-purpose):
  description: "ITC Negotiation Round [1|3|5] - Coding Agent: Task [N]"
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

    ## Testing Agent's Amendments (Rounds 3 and 5 — omit entirely on Round 1)

    [Paste the testing agent's most recent response here (Round 2 for Round 3 dispatch, Round 4 for Round 5 dispatch).
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

    When dispatched for solution-level ITC negotiation (after all tasks complete):

    **Required inputs** (all three must be loaded before drafting):
    1. The project's `<topic>-journeys.yaml` — pasted in full at the end of this prompt.
    2. `skills/subagent-driven-development/testing-strategies.md` — pasted in full at the end of this prompt.
    3. The actual implementation — scan the codebase for entry points, protected routes, and test files.

    Your output is a **coverage matrix**: rows are journey IDs from journeys.yaml, columns are strategy IDs from testing-strategies.md. Every cell MUST be either a `scenario` (with `command` and `assertion_shape`) or an `na` with a non-empty justification string. Do NOT invent strategy IDs — use the canonical IDs from the playbook verbatim.

    Use this YAML format:

    ```yaml
    solution: "[feature or project name]"
    tasks_covered: [1, 2, 3]

    implementation_summary:
      entry_points:
        - "[HTTP method] [route or component path]"
      protected_routes:
        - "[HTTP method] [route]"

    test_contract:
      tiers_required: [e2e, full_suite]
      rationale: "[one sentence: all tasks complete, verify full user journey end-to-end via coverage matrix]"

      e2e:
        required_services:
          - "[what must be running — e.g., 'backend API on port 3001', 'database seeded', 'frontend built']"

        coverage_matrix:
          J1:
            happy_path:
              scenario:
                command: "[exact runnable command — e.g., npx playwright test tests/e2e/flow.spec.ts --grep @J1-happy]"
                assertion_shape: "[concrete deep property — not just HTTP 200 or element visible]"
            negative_path:
              scenario:
                command: "..."
                assertion_shape: "..."
            state_persistence:
              scenario:
                command: "..."
                assertion_shape: "..."
            feature_interaction:
              scenario:
                command: "..."
                assertion_shape: "..."
            auth_boundary:
              scenario:
                command: "..."
                assertion_shape: "..."
          J2:
            happy_path:    { scenario: { command: "...", assertion_shape: "..." } }
            negative_path: { scenario: { command: "...", assertion_shape: "..." } }
            state_persistence: { na: "Journey is a single atomic request with no multi-step state to persist." }
            feature_interaction: { scenario: { command: "...", assertion_shape: "..." } }
            auth_boundary: { na: "Journey is fully public; no authentication or user-scoped data." }

      full_suite:
        command: "[exact command to run full test suite — e.g., npm test]"
        rationale: "Catch any cross-task regression not visible in individual task ITCs"

    acceptance_criteria:
      e2e: every scenario cell PASS; every na cell has non-empty justification
      full_suite: all pass
      no_regression: true

    sign_off:
      coding_agent: ✅
      testing_agent: (pending)
    ```

    **Rules for the matrix**
    - Every journey in journeys.yaml MUST be a row.
    - Every strategy ID in testing-strategies.md MUST be a column for every row.
    - Every cell = `scenario` or `na`. No empty cells.
    - `assertion_shape` is required prose; shallow shapes ("returns 200", "element visible" alone) will be rejected by the testing-agent.
    - `na` justification must cite a legitimate reason: structurally inapplicable, covered by another cell, or explicitly deferred. Lazy na ("too hard", "not critical", one-word) will be rejected.

    ## Journeys.yaml (paste in full)

    [Dispatcher: paste the full contents of `docs/superpowers/specs/<date>-<topic>-journeys.yaml` here]

    ## Testing Strategies Playbook (paste in full)

    [Dispatcher: paste the full contents of `skills/subagent-driven-development/testing-strategies.md` here]

    End your response with ✅ to confirm you have produced your proposal.
```
