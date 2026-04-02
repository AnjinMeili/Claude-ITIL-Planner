---
name: prereq-chain
description: |
  Use when executing a multi-step workflow where each phase produces outputs
  consumed by the next phase. Enforces ordered phase execution by verifying
  completion criteria before each phase transition, preventing compounding
  errors caused by premature advancement. Examples:

  <example>
  Context: User describes a multi-phase project flow with clear sequential dependencies between phases.
  user: "We have a design phase, then implementation, then testing, then deployment — each one depends on the last"
  assistant: "I'll set up a prereq-chain for this workflow. Before we start the first phase, I'll define the completion criteria for each gate so that phase transitions are enforced and no phase can start until the prior one has provably met its criteria."
  <commentary>
  Four ordered phases with explicit sequential dependency is the textbook prereq-chain trigger. The user has described all three qualifying conditions: three or more phases, phase outputs consumed by next phase, and defects in earlier phases increase downstream cost.
  </commentary>
  </example>

  <example>
  Context: User wants to establish workflow discipline before any code is written.
  user: "Before we start coding, set up the phase gates"
  assistant: "I'll invoke prereq-chain to define and document the gates now. Setting up the chain before execution begins is the correct moment — completion criteria defined after a phase starts are shaped by what the work produced, not what the phase was supposed to achieve."
  <commentary>
  "Before we start coding" signals intentional pre-execution setup. The user understands the value of gates and is explicitly requesting them before work begins — which is the only valid time to define them. This is also a clear prereq-chain trigger because it implies a multi-phase workflow exists.
  </commentary>
  </example>
version: 1.0.0
---

# prereq-chain

A verification gate between ordered workflow phases. Before any phase N+1
begins, the gate checks that phase N has met its defined completion criteria.
If criteria are met, the phase is marked CLEARED and execution proceeds. If
criteria are unmet, execution is BLOCKED and the specific unmet criteria are
surfaced.

The core insight: in multi-step workflows, the primary cause of compounding
errors is moving to step N before step N-1 is truly complete. Prerequisites
skipped under time pressure or ambiguous "done" definitions create debt that
surfaces as harder failures later — often in phases far removed from the
actual source.

---

## When to invoke

Invoke prereq-chain for any workflow that meets all three conditions:

1. Three or more ordered phases
2. Each phase produces artifacts or decisions consumed by the next phase
3. Defects in an earlier phase meaningfully increase the cost of correction
   in later phases

**Examples:** design → implement → test → deploy pipelines; specification →
generation → review → merge flows; research → plan → execute → verify agent
workflows.

## When NOT to invoke

Do not invoke prereq-chain for:

- **Parallel workflows** where phases are independent and can run
  concurrently without dependency
- **Single-phase tasks** where there is nothing to gate between
- **Exploratory or discovery work** where phases are not predefined and
  outputs are not consumed by successor phases

---

## Completion criteria types

Every phase in a prereq-chain must have its completion criteria defined
before the phase begins. Four criteria types cover the full space of phase
completion:

### Artifact criteria

Specific files, documents, or outputs must exist at a defined location.
Artifact criteria are binary: the artifact either exists or it does not.

Examples: SCOPE.md exists and is committed; test results file written to
`/results/`; implementation code present with no placeholder stubs.

### Quality criteria

Outputs must meet a defined standard — a score, a passing threshold, or a
reviewable signal. Quality criteria require a measurable check, not a
judgment call.

Examples: plan-critic score ≥ 7.5; all tests pass with no failures; linter
exits with zero warnings; coverage meets the threshold defined in SCOPE.md.

### Decision criteria

A specific decision must be recorded before the phase is considered complete.
Decision criteria capture not just what was decided but why the alternatives
were not chosen. The rationale is load-bearing: it prevents relitigating the
same decision in later phases.

Examples: architectural decision recorded with rationale and alternatives
rejected; deviation from the plan documented with justification; failing tests
given a recorded disposition (fix, defer, or won't fix with rationale).

### Confirmation criteria

Explicit confirmation from a specific party is required. Confirmation criteria
make implicit approvals explicit and traceable.

Examples: stakeholder has acknowledged the plan; code review completed (or
self-review checklist completed and signed); go/no-go decision recorded before
deployment.

---

## Defining a prereq-chain for a workflow

Define the chain before execution begins. Retroactively defining criteria
after a phase starts is an anti-pattern (see references).

**Steps:**

1. **Name each phase.** Use a consistent verb-noun form: Design, Plan,
   Implement, Test, Deploy. Each phase name should unambiguously describe
   what work occurs within it.

2. **Define completion criteria for each phase.** For each phase, specify
   one or more criteria across the four types. At minimum, every phase should
   have an artifact criterion (something exists) and either a quality or
   decision criterion (something has been verified or decided).

3. **Define the BLOCKED signal.** For each phase, specify what to emit if
   criteria are unmet: which criteria failed, what evidence is missing, and
   what must happen before the gate can re-check.

4. **Document the chain as a numbered list with criteria explicit.** The
   chain is a planning artifact, not a mental model. Write it down before
   execution starts.

**Example chain:**

```
1. Design
   Artifact: SCOPE.md committed to repository
   Quality: plan-critic score ≥ 7.5
   Decision: architectural approach recorded with rationale
   Confirmation: user acknowledged scope

2. Implement
   Artifact: code exists, tests exist, no unresolved TODO/FIXME
   Quality: all tests pass, linter clean
   Decision: any deviations from SCOPE.md documented

3. Test
   Artifact: test results file with pass/fail summary
   Quality: coverage meets threshold defined in SCOPE.md
   Decision: all failing tests have recorded disposition
   Confirmation: results reviewed against SCOPE.md success criteria

4. Deploy
   Artifact: deployment runbook executed
   Quality: no open critical issues, rollback path documented
   Decision: go/no-go decision recorded
   Confirmation: explicit sign-off obtained
```

---

## The gate protocol

Before starting phase N+1, execute the following gate check against phase N:

1. **Enumerate the completion criteria** for phase N as defined in the chain.

2. **Check each criterion** against available evidence (files, test output,
   recorded decisions, confirmation records).

3. **Emit gate output** in the standard format:

```
PHASE [N] → [N+1] GATE: [CLEARED|BLOCKED]
[criterion type]: [evidence or specific unmet condition]
[criterion type]: [evidence or specific unmet condition]
```

**If CLEARED:** document the gate passage with evidence and proceed to phase
N+1.

**If BLOCKED:** emit the BLOCKED signal with every unmet criterion listed
explicitly. Do not proceed to phase N+1. Do not treat BLOCKED as advisory.

### Gate output format

```
PHASE 2 → 3 GATE: CLEARED
Artifact: implementation code present, tests present, no TODO/FIXME found
Quality: 47/47 tests pass, linter exits 0
Decision: deviation from SCOPE.md §3.2 documented in DECISIONS.md

PHASE 2 → 3 GATE: BLOCKED
Quality: 3 tests failing — src/auth_test.go:L44, L89, L102
Decision: no recorded disposition for failing tests
Required: resolve or record disposition for all failing tests before re-check
```

The gate output is a record, not a conversation. It should be terse, specific,
and reference exact evidence. Vague gate output defeats the purpose.

---

## Common mistakes

**Vague completion criteria.** "Phase is done when it feels complete" is not
a criterion. Every criterion must have a specific, checkable signal. If a
criterion cannot be checked without judgment, rewrite it until it can.

**Skipping gate checks under time pressure.** Time pressure is the condition
under which gate checks matter most. A skipped gate is not a shortcut — it
is deferred cost with interest. Phases blocked and unblocked honestly take
less total time than phases that compound errors downstream.

**Treating BLOCKED as optional.** BLOCKED means halt. It does not mean
"consider whether to proceed." A workflow that regularly overrides BLOCKED
signals does not have a prereq-chain — it has a checklist that gets ignored.

**Defining criteria after the phase starts.** Completion criteria defined
after work begins are shaped by what the work produced, not by what the phase
was supposed to achieve. Define criteria before the phase begins, as part of
the chain definition.

**Partial-credit CLEARED.** Marking a phase CLEARED when some criteria are
met and others are not is criteria washing. Every criterion must be met for
CLEARED. If a criterion proves wrong or inapplicable, update the chain
definition explicitly — do not silently skip the criterion.

---

## Integration with plan-critic

plan-critic and prereq-chain are complementary and operate at different
levels.

plan-critic verifies the plan itself before execution begins: it checks
whether the plan is internally consistent, addresses the stated goal, and
scores above the quality threshold. plan-critic is a gate on the plan
artifact.

prereq-chain verifies that execution follows the plan phase by phase. It
checks whether each phase produced the outputs the plan required before the
next phase begins. prereq-chain is a gate on execution progress.

The recommended sequence: apply plan-critic to score and approve the plan,
then apply prereq-chain to enforce phase transitions during execution.

---

## Integration with scope-anchor

scope-anchor produces SCOPE.md, which defines the success criteria for the
final phase of a workflow. prereq-chain enforces the intermediate gates that
lead to that final phase.

Specifically: SCOPE.md's success criteria become the completion criteria for
the final phase's gate check. prereq-chain's intermediate gates ensure that
the work arriving at the final phase is sound enough to be evaluated against
those success criteria.

When defining a prereq-chain, reference SCOPE.md explicitly in the final
phase's confirmation criterion: "results reviewed against SCOPE.md success
criteria."

---

## Worked Example

**Request:** Set up a prereq-chain for a 3-phase workflow: Design → Implement → Test

### Step 1 — Define the chain

Before any phase begins, document the chain with explicit criteria:

```
PREREQ-CHAIN: Design → Implement → Test

Phase 1: Design
  Artifact: SCOPE.md exists at project root and is committed
  Quality: plan-critic score ≥ 7.5 on the resulting PLAN.md
  Decision: technology choices recorded with rationale in DECISIONS.md
  Confirmation: user has acknowledged scope boundaries

Phase 2: Implement
  Artifact: implementation code present, unit tests present, no TODO/FIXME stubs
  Quality: all tests pass (npm test exits 0), linter clean (eslint exits 0)
  Decision: any deviations from PLAN.md documented in DECISIONS.md

Phase 3: Test
  Artifact: test-results.json written to /results/ with pass/fail summary
  Quality: code coverage ≥ threshold defined in SCOPE.md
  Decision: all failing tests have recorded disposition (fix, defer, or won't fix)
  Confirmation: results reviewed against SCOPE.md success criteria
```

### Gate check 1: Design → Implement (BLOCKED)

Design phase work has been attempted. Running the gate check before Implement begins:

```
PHASE 1 → 2 GATE: BLOCKED
Artifact: SCOPE.md — NOT FOUND at project root
Quality: PLAN.md present, plan-critic score 8.5/10 — CLEARED
Decision: DECISIONS.md present with technology rationale — CLEARED
Confirmation: user acknowledgment — CLEARED

Required: SCOPE.md must be created and committed before the Implement phase can begin.
Action: Run scope-anchor to produce SCOPE.md, then re-check this gate.
```

Implement phase does not start. scope-anchor is invoked. SCOPE.md is written and committed.

### Gate re-check 1: Design → Implement (CLEARED)

```
PHASE 1 → 2 GATE: CLEARED
Artifact: SCOPE.md present at project root, committed at a3f7c21 — CLEARED
Quality: PLAN.md plan-critic score 8.5/10 — CLEARED
Decision: DECISIONS.md with rationale for Express + PostgreSQL — CLEARED
Confirmation: user acknowledged scope at 14:32 UTC — CLEARED

Proceeding to Implement phase.
```

### Gate check 2: Implement → Test (CLEARED)

Implement phase complete. Running the gate check before Test begins:

```
PHASE 2 → 3 GATE: CLEARED
Artifact: src/ implementation present, src/__tests__/ present, no TODO/FIXME — CLEARED
Quality: 63/63 tests pass, eslint exits 0 — CLEARED
Decision: one deviation from PLAN.md (removed Redis caching step, rationale in DECISIONS.md §4) — CLEARED

Proceeding to Test phase.
```

Test phase begins with all implementation criteria verified. The BLOCKED signal at gate 1 prevented the Implement phase from starting on an incomplete Design foundation — exactly the compounding error prereq-chain is designed to prevent.

---

## References

- `references/completion-criteria-patterns.md` — detailed guide to defining
  completion criteria by task type, anti-patterns in criterion definition,
  and a fill-in-the-blank template for new workflows
