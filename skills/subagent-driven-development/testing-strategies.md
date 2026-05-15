# Testing Strategies Playbook

Canonical strategies the coverage matrix must address in every solution-level ITC. These column names in the coverage matrix MUST match section IDs below verbatim — do not invent new IDs inline; new strategies are introduced by appending sections to this file.

The testing-agent and coding-agent both walk this list for every journey listed in `<topic>-journeys.yaml` at solution-ITC negotiation time. Each journey × strategy cell is either a runnable scenario (with `command` and `assertion_shape`) or an explicit `na` with justification.

This list grows. New strategies are appended as new `##` sections. The coverage matrix requires a cell for every strategy currently in this file.

## happy_path

**When it applies:** Every journey. This is the baseline — the journey runs end-to-end with valid inputs from documented preconditions to the documented `expected_outcome`.

**Legitimate N/A reasons:** None. Every journey must have a happy_path scenario.

**Example scenario shape:**
- command: `npx playwright test tests/e2e/<journey>.spec.ts --grep @happy`
- assertion_shape: "Journey completes all steps; final DB state matches expected_outcome; UI shows success state; side-effects (emails, notifications) recorded."

**Common shallow mistakes:** Asserting only HTTP 200 or only element visible; skipping the DB-state check; not verifying side effects like emails actually sent.

## negative_path

**When it applies:** Any journey where an input is validated, an external service can fail, or a timeout is possible. In practice: nearly every journey.

**Legitimate N/A reasons:**
- Journey has no inputs and no external dependencies (rare; must cite specific steps).
- Negative-path coverage is fully subsumed by happy_path's assertions (e.g., an idempotent public read-only journey with no inputs).

**Example scenario shape:**
- command: `npx playwright test tests/e2e/<journey>.spec.ts --grep @negative`
- assertion_shape: "Invalid input at step X yields documented error response; no partial DB writes; no stale client state; user sees clear recovery path."

**Common shallow mistakes:** Only testing one negative case (e.g., just "invalid email"); asserting only status code without checking partial-write protection; missing service-failure simulation.

## state_persistence

**When it applies:** Journeys with 2+ steps, or any journey where intermediate state must survive navigation, reload, or session change.

**Legitimate N/A reasons:**
- Journey is a single atomic request with no client-side state to persist.

**Example scenario shape:**
- command: `npx playwright test tests/e2e/<journey>.spec.ts --grep @persistence`
- assertion_shape: "After reload mid-journey at step N, journey resumes from a valid state; authenticated session intact; partial data preserved or cleanly cleared per contract."

**Common shallow mistakes:** Only testing reload at the final step; not checking logout-login continuity; not asserting the specific state preserved.

## feature_interaction

**When it applies:** Any journey whose `feature_interactions` tag overlaps with another journey's tags.

**Legitimate N/A reasons:**
- Journey shares no `feature_interactions` tags with any other journey (truly isolated feature).

**Example scenario shape:**
- command: `npx playwright test tests/e2e/<journey>.spec.ts --grep @after-<other-journey-id>`
- assertion_shape: "Running journey J_other immediately before/after this journey: both end states correct; no shared state corruption; tag-overlapping features coexist."

**Common shallow mistakes:** Choosing an interacting journey at random rather than by shared `feature_interactions` tag; not running both journeys to completion; not asserting both end states.

## auth_boundary

**When it applies:** Any journey with authenticated steps, protected routes, or user-scoped data.

**Legitimate N/A reasons:**
- Journey is fully public; no authentication, no user-scoped data, no tenancy.

**Example scenario shape:**
- command: `npx playwright test tests/e2e/<journey>.spec.ts --grep @unauthorized`
- assertion_shape: "Unauthenticated actor hitting protected steps receives 401 or 403 (not 200); wrong-tenant actor cannot read or mutate data belonging to another tenant; no sensitive data in error response."

**Common shallow mistakes:** Testing only 'no token' but not 'wrong tenant'; asserting only response code without verifying no data leakage; skipping mutation endpoints and only testing reads.
