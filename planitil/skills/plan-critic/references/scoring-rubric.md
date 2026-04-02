# Plan Critic Scoring Rubric

Detailed scoring guide for each of the five plan-critic dimensions. Use this reference when the three-sentence rationale in the scorecard is not sufficient to determine a score, or when constructing revision instructions for a failing plan.

---

## Score semantics

Each dimension uses a three-level scale:

- **0 — Clear failure:** The dimension is absent from the plan, or the plan actively violates it. This is not a matter of degree; the problem is structural.
- **1 — Partial:** The dimension is addressed, but with notable gaps. The executor will encounter friction here. The plan is not wrong, but it is incomplete or inconsistent on this dimension.
- **2 — Solid:** The dimension is well-addressed. No significant gaps. An executor can follow this section of the plan without improvising.

A score of 2 does not mean perfect — it means the plan is adequate to proceed. A score of 0 means the plan must be revised before execution regardless of scores on other dimensions.

---

## Dimension 1: Goal Alignment

### Signal questions

1. Can every task in the plan be traced to the stated goal in one sentence or less?
2. Are there tasks that exist because they are "good practice" rather than because the goal requires them?
3. If a task were removed, would the goal still be achievable? (If yes and nothing else depends on it, the task may be misaligned.)

### Score 0 example

Plan excerpt:
> "Phase 1: Refactor authentication module. Phase 2: Add OAuth support. Phase 3: Update the user profile page layout. Phase 4: Write API documentation."

Goal stated: "Enable third-party OAuth login."

Explanation: Phase 3 (profile page layout) has no traceable connection to the OAuth goal. Phase 4 (API documentation) is potentially valid but would need to be scoped to the OAuth API specifically — as written, it covers all API documentation, which extends beyond the goal. Neither phase is justified by the stated goal. Score: 0.

### Score 2 example

Plan excerpt:
> "Phase 1: Add OAuth provider configuration (required to support Google and GitHub as login providers). Phase 2: Implement callback handler and token exchange (core OAuth flow — required to complete login). Phase 3: Update session creation to accept OAuth-sourced identity (required to persist the login state). Phase 4: Add integration tests for OAuth happy path and token expiry (required to verify the goal is met)."

Goal stated: "Enable third-party OAuth login."

Explanation: Every phase carries an explicit goal trace. The parenthetical reason for each phase answers "why does this serve the goal." Score: 2.

---

## Dimension 2: Completeness

### Signal questions

1. Could an executor follow this plan from start to finish without making an undocumented decision?
2. Does the plan end at the right place — with a verified, deployed, or confirmed outcome — or does it trail off before completion?
3. Does any single step contain compressed complexity ("configure as needed," "handle edge cases," "test and deploy")?

### Score 0 example

Plan excerpt:
> "Step 1: Build the feature. Step 2: Test and deploy."

Explanation: "Build the feature" is not a plan step — it is a goal restatement. "Test and deploy" compresses at minimum three phases (write tests, run tests, deploy to staging, validate staging, deploy to production) into a single placeholder. An executor encountering this plan must invent the entire implementation and all deployment steps. Score: 0.

### Score 2 example

Plan excerpt:
> "Step 1: Write the data transformer function with unit tests. Acceptance: all edge cases from the spec doc covered, tests pass. Step 2: Integrate the transformer into the pipeline at the ingestion layer. Acceptance: pipeline runs without regression on the existing test suite. Step 3: Deploy to staging. Acceptance: staging smoke test passes. Step 4: Deploy to production with feature flag disabled. Step 5: Enable feature flag for 5% of traffic. Monitor error rate for 30 minutes. Step 6: If error rate stable, ramp to 100%. If not, disable flag and open incident."

Explanation: Every step is atomic enough to execute, has an acceptance criterion, and the rollout sequence is fully specified. The executor does not need to invent any step. Score: 2.

---

## Dimension 3: Risk Surface

### Signal questions

1. Does the plan include any irreversible operation (data deletion, schema migration, external API call, production deployment, file overwrites)?
2. For each irreversible operation: is there a rollback step or a mitigation strategy present in the plan?
3. Does the plan identify the most likely failure mode and specify what to do when it occurs?

### Score 0 example

Plan excerpt:
> "Step 3: Run the database migration to add the new columns. Step 4: Update all existing rows to populate the new columns. Step 5: Drop the old columns."

Explanation: Steps 4 and 5 are irreversible. Step 4 overwrites data in place — if the migration logic is wrong, records are corrupted. Step 5 drops columns permanently. No backup step precedes step 3, no validation step exists between 4 and 5, and no rollback path is specified if step 4 produces incorrect data. Score: 0.

### Score 2 example

Plan excerpt:
> "Step 3: Take a snapshot of the affected tables. Step 4: Run the database migration to add the new columns. Step 5: Update all existing rows in batches of 1,000 — log the row count before and after. Step 6: Validate row count parity and spot-check 20 randomly sampled rows. If validation fails, halt and restore from step 3 snapshot. Step 7: After validation passes, drop the old columns. Rollback if needed: restore from step 3 snapshot and re-run the inverse migration."

Explanation: The snapshot precedes irreversible work. Batch processing limits blast radius. Validation with an explicit failure path gates the irreversible column drop. A named rollback procedure exists. Score: 2.

---

## Dimension 4: Scope Fit

### Signal questions

1. Does every task in the plan correspond to an item on the SCOPE.md in-scope list?
2. Are there tasks in the plan that the SCOPE.md out-of-scope list explicitly excludes?
3. If SCOPE.md does not exist: is the plan operating on stated or unstated assumptions about its boundaries?

### Automatic penalty

If no SCOPE.md exists and `scope-anchor` has not been run in the current session: subtract 1 from the raw score for this dimension before recording it. A plan without an anchoring scope document is assumed to be operating on unstated assumptions. This penalty applies even if the plan appears internally consistent — the issue is not the plan's quality but the absence of an external reference to validate it against.

### Score 0 example

SCOPE.md in-scope list: "Migrate user authentication from session tokens to JWTs."

Plan excerpt includes: "Phase 4: While migrating auth, update the user profile API to use the new user ID format. Phase 5: Refactor the notification service to use the auth client we're building."

Explanation: Phase 4 extends scope to a different API without justification in SCOPE.md. Phase 5 includes the notification service, which was not mentioned in scope. Both represent silent scope expansion. The plan is doing more than what was agreed. Score: 0.

### Score 2 example

SCOPE.md in-scope list: "Migrate user authentication from session tokens to JWTs. Out of scope: user profile API changes, notification service, admin panel."

Plan excerpt: "Phase 1: Issue JWT on login. Phase 2: Validate JWT on protected endpoints. Phase 3: Remove session token issuance. Phase 4: Update tests to use JWT fixture."

Explanation: Every phase corresponds directly to the migration scope. Nothing in the plan touches the explicitly excluded services. Score: 2.

---

## Dimension 5: Anti-Pattern Presence

### Signal questions

1. Do any tasks in the plan use "while we're here," "since we're touching this anyway," or "might as well" language?
2. Does the plan address a problem different from the one stated in the goal — would it be a perfect plan for a different task?
3. Does the plan omit any verifiable completion condition, leaving success undefined?

### Named anti-patterns to check

**Scope Creep by Analogy:** Tasks that are added because they are adjacent to the work, not because the goal requires them. Signal phrase: "while we're here" or "since we're already in this file."

**Orphaned Goal:** The plan would be appropriate for a different goal than the one stated. The plan is internally coherent but addresses the wrong problem. Often occurs when a plan is copied or adapted without updating the goal section.

**Single-Step Compression:** A phase that contains genuine complexity is written as a single step with no sub-steps, hiding the work inside it. Signal: a step name that is actually a goal ("implement the feature," "handle the errors," "write the tests").

**Missing Rollback:** Irreversible operations appear without a named undo path. Overlaps with Risk Surface — both dimensions penalize this. An irreversible step with no rollback path scores 0 on Risk Surface and reduces Anti-Pattern to at most 1.

**Success Criteria Omission:** The plan reaches its final step without stating how the executor knows the work is done. The final step may be "deploy to production" with no acceptance check, no smoke test, and no confirmation step.

### Score 0 example

Plan excerpt:
> "Phase 3: Implement the feature. While we're in the auth module, clean up the legacy password reset flow — it's been on the backlog and we'll already be touching auth. Phase 4: Deploy. Done."

Explanation: Phase 3 contains Scope Creep by Analogy ("while we're in the auth module") with an explicit reference to backlog work that was not in the goal. Phase 4 is Single-Step Compression (no deploy sub-steps) and Success Criteria Omission (no acceptance condition). Two catalog anti-patterns are structurally present. Score: 0.

### Score 2 example

Plan excerpt:
> "Phase 1: Add the JWT issuance function. Complete when: unit tests pass and the function matches the interface spec in `docs/auth-interface.md`. Phase 2: Integrate JWT into the login endpoint. Complete when: integration test passes and no regression in existing auth tests. Phase 3: Deploy to staging. Complete when: staging smoke test passes and QA signs off. Phase 4: Deploy to production. Complete when: production health check passes and error rate is stable for 15 minutes."

Explanation: No "while we're here" language. Every phase has an explicit completion condition. No step is compressed. The plan matches the stated goal. Score: 2.

---

## Passing threshold rationale

The passing threshold is 7.5 out of 10.

At 7.5, the minimum score distribution is: four dimensions at 2 and one dimension at 0, or some combination that sums to at least 7.5 (e.g., four 2s and one 0 = 8; three 2s and two 1s = 8; four 2s and one partial-but-not-zero = 8 or 9). A score of exactly 7.5 is not achievable with integer scores — the threshold is functionally "8 or above."

The threshold is designed to allow one dimension to be partial (1 point) while requiring the other four to be solid (2 points each), for a minimum passing score of 2+2+2+2+1 = 9. Alternatively, it allows three solid dimensions and two partials: 2+2+2+1+1 = 8.

A plan scoring below 7.5 has at least two weak dimensions. Two structural gaps in a plan do not add — they compound. A plan with both a Goal Alignment failure and a Completeness failure will encounter misaligned work and improvised steps. A plan with both a Risk Surface failure and an Anti-Pattern failure will encounter unexpected irreversible actions and predictable failure modes. Below 7.5 is a genuine structural problem, not a margin call.

---

## Common patterns in failing plans

### Pattern: Task list without goal traceability

Appearance: The plan is a numbered list of tasks with no stated goal and no per-task justification. Every task is stated as an imperative ("do X") with no connection to a stated outcome.

Failure: Goal Alignment = 0.

Revision instruction template:
> "Add a goal statement at the top of the plan. Then, for each task, add a one-sentence justification that traces it to the stated goal. Remove any task that cannot be traced."

---

### Pattern: "Test and deploy" as a single step

Appearance: The plan's final one or two steps are "test," "deploy," or "test and deploy" with no sub-steps, no acceptance criteria, and no environment specification.

Failure: Completeness = 0 or 1, Anti-Pattern (Single-Step Compression) = 0 or 1.

Revision instruction template:
> "Expand '[step name]' into discrete sub-steps. At minimum: write what tests must be run, what environment they run in, what the acceptance condition is, what deploy targets are involved (staging, production), and what the validation step is after each deploy. Add a final acceptance step stating how the executor knows the work is complete."

---

### Pattern: No mention of failure paths

Appearance: The plan proceeds linearly from start to finish. No step mentions what happens if the previous step fails. No rollback steps exist. No monitoring or validation steps follow irreversible operations.

Failure: Risk Surface = 0 or 1, Anti-Pattern (Missing Rollback) = 0 or 1.

Revision instruction template:
> "Identify all irreversible operations in the plan (data writes, migrations, deletes, external calls, production deployments). For each: add a step immediately preceding it to create a recovery artifact (snapshot, backup, export). Add a step immediately following it to validate the outcome. Add a rollback step specifying exactly how to undo the operation if the validation fails."

---

### Pattern: Plan written before SCOPE.md exists

Appearance: The plan references no SCOPE.md. The scope of the plan is implied by its task list rather than stated in a separate document.

Failure: Scope Fit = 0 or 1 (with automatic -1 penalty applied). May also trigger Goal Alignment issues if the scope was never agreed.

Revision instruction template:
> "Run scope-anchor before continuing with this plan. No plan scoring can complete for Scope Fit until a SCOPE.md exists. Once SCOPE.md is written and confirmed, re-run plan-critic and score Scope Fit against it."

---

### Pattern: "While we're here" tasks

Appearance: One or more tasks in the plan are prefaced with "while we're here," "since we're already touching this," "might as well," or "this has been on the backlog." These tasks are not required by the goal.

Failure: Anti-Pattern (Scope Creep by Analogy) = 0. May also affect Scope Fit = 0 or 1 if the out-of-scope items are explicitly excluded in SCOPE.md.

Revision instruction template:
> "Remove the following tasks from the plan: [list tasks with 'while we're here' framing]. If these tasks are genuinely needed, add them to the backlog and create a separate work item. If they are required to achieve the current goal, rewrite them with an explicit goal trace and update SCOPE.md to include them before re-scoring."
