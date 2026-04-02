---
name: plan-critic
description: |
  Use when any plan has been written and before execution begins. Scores the plan
  against five structural dimensions using an evaluator-optimizer loop: score →
  fail → revise → re-score, until the plan passes a 8/10 threshold or escalates
  to the user after two revision cycles. Prevents execution from starting on
  structurally deficient plans. Designed to run after scope-anchor and
  failure-modes have already run. Examples:

  <example>
  Context: User has just finished writing a plan and explicitly asks for a review before proceeding.
  user: "I've written the plan, can you review it"
  assistant: "I'll invoke plan-critic to score it before we proceed. Let me read the plan and run it through the five scoring dimensions — Goal Alignment, Completeness, Risk Surface, Scope Fit, and Anti-Pattern Presence — then emit a scorecard."
  <commentary>
  An explicit user request to review a plan is the clearest possible trigger for plan-critic. The user is asking for quality assurance before execution, which is exactly what the evaluator-optimizer loop provides.
  </commentary>
  </example>

  <example>
  Context: The plan phase of a multi-phase workflow has just completed automatically as part of a larger pipeline.
  user: "[workflow advances automatically after plan-phase output]"
  assistant: "Plan phase complete. Invoking plan-critic automatically before transitioning to execution. Scoring the plan now against the five dimensions."
  <commentary>
  plan-critic should run automatically at plan→execution phase transitions, not only when explicitly requested. This is the key integration point with prereq-chain: the phase gate should not clear unless plan-critic has passed.
  </commentary>
  </example>
version: 1.0.0
argument-hint: [plan-file-or-description]
---

# plan-critic

An evaluator-optimizer loop that gates execution on plan quality. No plan proceeds to execution until it scores 8 or higher across five structural dimensions. When a plan fails, the critic produces specific revision instructions per failing dimension and re-scores after revision — up to two cycles before escalating to the user.

This skill implements the Anthropic evaluator-optimizer pattern at the planning level: generate plan → critic scores → if below threshold, revise → repeat. It operationalizes the scenario-based evaluation approach: each dimension is a named criterion with an explicit scoring rubric, not a vague quality judgment.

---

## When to invoke

Invoke after:

- Any plan has been written (GSD PLAN.md artifacts, implementation plans, architecture plans, work breakdowns)
- A plan has been revised mid-execution and the revision is significant (scope change, new phases, resequencing)
- A plan has been imported or handed off from another session or agent

Do not invoke for:

- Trivial tasks with one to three steps where the "plan" is implicit in the request
- Tasks where the entire work is a single atomic action (rename a file, fix a typo)
- Plans that have already passed critic in the current session — re-scoring an already-passing plan wastes cycles

**Integration sequencing:** `plan-critic` works best after `scope-anchor` (so the Scope Fit dimension has a SCOPE.md to check) and after `failure-modes` (so known anti-patterns are already cataloged before scoring). Running `plan-critic` without a prior `scope-anchor` run will deduct one point automatically from the Scope Fit dimension.

---

## The five scoring dimensions

Each dimension scores 0, 1, or 2 points. Maximum score is 10. Passing threshold is 8.

The threshold allows one dimension to be partial while requiring solidity across the rest. A plan scoring below 8 has at least two weak dimensions — structural problems that will manifest during execution.

### 1. Goal Alignment (0–2)

Does every task in the plan trace back to the stated goal? Every step should exist because the goal requires it. Steps that exist because they are "good practice," "while we're here," or "related" without direct traceability to the goal are misalignment signals.

Score 2 when every step in the plan connects to the goal either directly or through a clear dependency chain. Score 1 when most steps connect but one or two are loosely justified. Score 0 when tasks are not traced to the goal at all, or when the plan addresses a different goal than the one stated.

### 2. Completeness (0–2)

Are all steps present? A complete plan is one where an executor can follow it without improvising. Gaps — missing steps between phases, hand-waves like "configure as needed," "test and deploy," or "handle edge cases" as single atomic steps — signal that the plan will require in-flight decisions that should have been made upfront.

Score 2 when the plan can be followed without gaps and all expected phases are present. Score 1 when most steps are defined but one or two require the executor to invent detail. Score 0 when the plan skips entire phases (e.g., no test phase, no rollout phase) or uses placeholder language for critical steps.

### 3. Risk Surface (0–2)

Are risky or irreversible steps identified and mitigated? Risk Surface is not about predicting every failure — it is about whether the plan acknowledges the steps where something going wrong is difficult to reverse, and provides either a mitigation or a rollback path.

Score 2 when irreversible operations are called out, mitigations or rollback steps are specified, and the plan includes a recovery path for the most likely failure mode. Score 1 when risk is partially acknowledged — some dangerous steps are noted but without a concrete mitigation. Score 0 when no failure paths appear anywhere in the plan, especially when the plan includes destructive operations (deletes, migrations, schema changes, external API calls).

### 4. Scope Fit (0–2)

Does the plan stay within SCOPE.md boundaries? Every task in the plan should correspond to something in the SCOPE.md in-scope list. Tasks that address out-of-scope items, or that expand scope without explicitly updating SCOPE.md, are violations.

**Automatic penalty:** If no SCOPE.md exists and `scope-anchor` has not been run in the current session, subtract 1 from this dimension's score before applying the rubric. A plan written without anchored scope is operating on unstated assumptions.

Score 2 when every task maps to a SCOPE.md in-scope item and no task addresses an out-of-scope item. Score 1 when the plan is mostly aligned but includes one or two tasks that extend scope without acknowledgment. Score 0 when the plan contains significant out-of-scope work, or when no SCOPE.md exists.

### 5. Anti-Pattern Presence (0–2)

Do any entries from the failure-modes catalog apply to this plan? This dimension requires consulting the `failure-modes` skill's planning catalog. The most common plan-level anti-patterns that should trigger deductions:

- **Scope Creep by Analogy** — the plan includes tasks prefixed with "while we're here" or "since we're touching this anyway"
- **Orphaned Goal** — the plan contains tasks that would be appropriate for a different goal than the one stated
- **Missing Rollback** — irreversible operations appear without a corresponding undo path (overlaps Risk Surface; both should penalize)
- **Single-Step Compression** — a complex phase is compressed into one step, hiding the work inside it
- **Success Criteria Omission** — the plan has no verifiable completion condition

Score 2 when none of the cataloged failure modes appear in the plan. Score 1 when one or two minor patterns are present but do not invalidate the plan. Score 0 when a catalog failure mode is structurally present — it is not an edge case, it is the shape of the plan.

---

## The evaluation loop

### Step 1 — Read the plan

If invoked with a file path argument: read that file. If invoked with no argument: look for a PLAN.md in the current working directory or the most recently referenced plan in the conversation context. If no plan is found, ask for one before proceeding.

### Step 2 — Score each dimension

For each of the five dimensions, assign a score of 0, 1, or 2. Write one to three sentences of rationale for the score. The rationale must be specific: cite the plan section or step that caused the score, not a general observation.

### Step 3 — Sum and verdict

Sum the five scores. Apply the verdict:

- **Score ≥ 8: PASS** — emit the scorecard and a brief summary of any dimension scoring 1 (partial issues worth watching in execution). Execution may proceed.
- **Score < 8: FAIL** — emit the scorecard and revision instructions. Revision instructions are dimension-specific: one instruction block per failing or partial dimension, stating exactly what must change.

### Step 4 — Revise and re-score (on FAIL)

Apply the revision instructions to the plan. Re-score from scratch — do not carry over scores from the prior round. If the revised plan passes, emit PASS. If it fails again, apply revision instructions once more.

After two failed revision cycles, escalate: emit the final scorecard, note that two revision cycles were attempted, and ask the user how to proceed. Do not attempt a third revision cycle autonomously.

---

## Output format

### Scorecard (emitted on every scoring pass)

```
## Plan Critic Scorecard

| Dimension         | Score | Rationale                          |
|-------------------|-------|------------------------------------|
| Goal Alignment    | N/2   | [specific rationale]               |
| Completeness      | N/2   | [specific rationale]               |
| Risk Surface      | N/2   | [specific rationale]               |
| Scope Fit         | N/2   | [specific rationale]               |
| Anti-Pattern      | N/2   | [specific rationale]               |
| **Total**         | N/10  |                                    |

**Verdict: PASS / FAIL**
```

On PASS, append:

```
Execution may proceed. Watching: [list any dimensions that scored 1 with a one-line note]
```

On FAIL, append one block per failing or partial dimension:

```
## Revision Required — [Dimension Name]

[Specific instruction stating what must be added, removed, or changed in the plan]
```

---

## Common revision instruction patterns

**Goal Alignment failure:** Identify which tasks lack goal traceability. State which tasks to remove or rewrite with an explicit goal linkage.

**Completeness failure:** Identify the gap. State which phase or step is missing and what it must contain at minimum. Name the hand-wave language that triggered the deduction.

**Risk Surface failure:** Identify the irreversible operations. State that a rollback step or mitigation must be added for each. Specify the failure mode to mitigate (data loss, API rate limit, migration lock, etc.).

**Scope Fit failure:** Identify the out-of-scope tasks by name. State that they must either be removed or that SCOPE.md must be updated to include them before proceeding.

**Anti-Pattern failure:** Name the specific anti-pattern (use the failure-modes catalog name). State the section of the plan where it appears. State the structural change needed to eliminate it.

---

## Design rationale

This skill is an instance of the Anthropic evaluator-optimizer pattern: a separate evaluation step that grades output against explicit criteria before that output is used downstream. The pattern is most valuable at the planning stage because plans are cheap to revise and expensive to execute wrong.

The five dimensions are not arbitrary — they correspond to the five most common structural failure modes in AI-generated plans:

1. Plans that solve the wrong problem (Goal Alignment)
2. Plans that omit phases because they seemed obvious (Completeness)
3. Plans that don't account for failure (Risk Surface)
4. Plans that expand scope silently (Scope Fit)
5. Plans that repeat known failure patterns (Anti-Pattern Presence)

The 8/10 threshold is intentionally forgiving on any one dimension while requiring overall solidity. A plan that is perfect on four dimensions and weak on one can still pass. A plan with two weak dimensions cannot — because two structural gaps compound during execution.

## Evaluator agent

The scoring step (Step 2) is performed by the `plan-critic-evaluator` agent — a
separate Haiku-class agent with Read-only tool access. This architectural separation
ensures the model that wrote or revised the plan is not also the sole judge of its
quality. The evaluator agent receives the plan file path and returns a structured
scorecard. plan-critic orchestrates the loop; plan-critic-evaluator does the scoring.

---

## Worked Example

**Input plan (weak, 4 tasks):**

```markdown
## Plan: Add user authentication

1. Create login form
2. Add backend endpoint
3. Test and deploy
```

### Round 1 scorecard

```
## Plan Critic Scorecard

| Dimension         | Score | Rationale                                                                          |
|-------------------|-------|------------------------------------------------------------------------------------|
| Goal Alignment    | 2/2   | All three tasks relate to the stated goal of adding user authentication.           |
| Completeness      | 0/2   | Step 3 "Test and deploy" compresses two distinct phases into one atomic step with  |
|                   |       | no sub-steps. No session management, password hashing, or error handling steps     |
|                   |       | appear. The plan cannot be followed without invention.                             |
| Risk Surface      | 1/2   | No rollback path for the deployment step. No mention of migration for the user     |
|                   |       | table schema. Partially acknowledged by grouping test before deploy.               |
| Scope Fit         | 2/2   | SCOPE.md confirmed authentication as in scope. No out-of-scope tasks detected.    |
| Anti-Pattern      | 1/2   | Single-Step Compression is present in step 3. Not plan-invalidating but requires  |
|                   |       | expansion before execution.                                                        |
| **Total**         | 6/10  |                                                                                    |

**Verdict: FAIL**
```

### Revision instructions

```
## Revision Required — Completeness

Step 3 ("Test and deploy") must be expanded into at minimum:
- 3a. Write unit tests for login form validation
- 3b. Write integration tests for the backend authentication endpoint
- 3c. Run full test suite; confirm all pass
- 3d. Deploy to staging; smoke test login flow end-to-end
- 3e. Deploy to production; monitor error rate for 15 minutes post-deploy

Additionally, add a step between 1 and 2 covering: password hashing strategy (bcrypt rounds), session token generation, and user table schema migration.

## Revision Required — Anti-Pattern (Single-Step Compression)

Step 3 is a Single-Step Compression anti-pattern. See Completeness revision above for the required expansion. Once expanded, this dimension will re-score at 2.
```

### Round 2 — revised plan (excerpt)

```markdown
## Plan: Add user authentication

1. Create login form with client-side validation (email format, required fields)
2. Add user table migration: id, email (unique), password_hash, created_at
3. Implement password hashing with bcrypt (12 rounds)
4. Add POST /auth/login endpoint — validates credentials, issues session token
5. Add session middleware to protect authenticated routes
6. Write unit tests for form validation and password hashing logic
7. Write integration tests for /auth/login (valid, invalid password, unknown email)
8. Run full test suite; all tests must pass before proceeding
9. Deploy to staging; smoke test login flow
10. If staging passes: deploy to production; monitor p95 error rate for 15 minutes
11. Rollback plan: if error rate exceeds baseline by >5%, redeploy previous release tag
```

### Round 2 scorecard

```
## Plan Critic Scorecard

| Dimension         | Score | Rationale                                                                    |
|-------------------|-------|------------------------------------------------------------------------------|
| Goal Alignment    | 2/2   | All 11 steps trace directly to the authentication goal.                      |
| Completeness      | 2/2   | Schema, hashing, endpoint, session, tests, staging, production, rollback     |
|                   |       | all present. Executor can follow without inventing steps.                    |
| Risk Surface      | 2/2   | Step 11 provides an explicit rollback with a concrete trigger condition.     |
| Scope Fit         | 2/2   | All steps within SCOPE.md boundary.                                          |
| Anti-Pattern      | 2/2   | Single-Step Compression resolved. No catalog patterns present.               |
| **Total**         | 10/10 |                                                                              |

**Verdict: PASS**

Execution may proceed. Watching: nothing — all dimensions at full score.
```

---

## References

- [Scoring rubric](references/scoring-rubric.md) — detailed scoring guide with signal questions, scored examples, and revision instruction templates per dimension
