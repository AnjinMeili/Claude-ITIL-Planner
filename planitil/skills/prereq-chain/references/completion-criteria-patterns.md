# Completion Criteria Patterns

A reference guide for defining prereq-chain completion criteria by task type.
Use this document when building a prereq-chain to ensure each phase has
criteria that are specific, checkable, and sufficient.

---

## Criteria by task type

### Design / Planning phases

Design and planning phases are complete when there is a durable artifact that
captures what will be built, a quality signal that the artifact was evaluated,
decisions that cannot be relitigated downstream, and confirmation that the
right parties have seen and acknowledged the plan.

**Artifact criteria**
- SCOPE.md or PLAN.md exists and is committed to the repository (or equivalent
  durable location — not a local scratch file)
- The artifact covers all sections required by the workflow definition (goals,
  constraints, success criteria, out-of-scope)

**Quality criteria**
- plan-critic score ≥ 7.5 (or the threshold defined for the workflow)
- All open questions in the plan have been resolved or explicitly parked with
  a stated resolution path

**Decision criteria**
- Architectural or approach decisions are recorded with rationale — not just
  "what was decided" but "why the alternatives were not chosen"
- Trade-offs are explicit: the decision record should contain at least one
  rejected alternative with a reason for rejection

**Confirmation criteria**
- Stakeholder or user has acknowledged the plan in a traceable way (message,
  comment, sign-off record)
- If the plan affects downstream agents or systems, those dependencies are
  notified or recorded

---

### Implementation phases

Implementation phases are complete when the code exists and is correct, tests
confirm it, and any drift from the plan is documented.

**Artifact criteria**
- Production code exists for all planned functionality
- Tests exist for all production code paths covered by the phase
- No unresolved TODO or FIXME comments remain — each must either be resolved
  or converted to a tracked item with an explicit deferral rationale

**Quality criteria**
- All tests pass with no failures
- Linter exits with zero warnings (or all warnings are suppressed with
  explicit inline justification — not blanket suppression)
- No new warnings introduced relative to the baseline at phase start
- If a coverage threshold was defined in SCOPE.md, coverage meets that
  threshold

**Decision criteria**
- Any deviations from the plan are documented with rationale — "what changed,
  why it changed, and what impact it has on downstream phases"
- If an originally planned approach was abandoned, the abandonment is recorded
  so later phases do not attempt to build on the abandoned approach

**Confirmation criteria**
- Code review completed (by another reviewer, or self-review checklist
  completed and signed if solo)
- If the implementation has external dependencies (APIs, infrastructure,
  third-party services), those dependencies have been verified as available
  and compatible

---

### Testing phases

Testing phases are complete when results are recorded, coverage is sufficient,
and every failure has a disposition.

**Artifact criteria**
- Test results file exists with a pass/fail summary at a defined path
- The results file is machine-readable or structured enough to be referenced
  by downstream phases (not just console output)

**Quality criteria**
- Coverage meets the threshold defined in SCOPE.md or in the phase definition
- All acceptance criteria listed in SCOPE.md have been exercised by at least
  one test
- Regression suite passes — no tests that were previously passing are now
  failing

**Decision criteria**
- Every failing test has a recorded disposition: fix (with timeline), defer
  (with rationale and risk acknowledgment), or won't fix (with rationale)
- A failing test with no disposition is an open question, not a completed
  phase

**Confirmation criteria**
- Test results have been reviewed against SCOPE.md success criteria — not
  just "tests pass" but "the passing tests cover what SCOPE.md required"
- If external stakeholders defined acceptance criteria, those criteria are
  checked against results explicitly

---

### Review / Deploy phases

Review and deploy phases carry the highest cost of error reversal. Criteria
here are stricter and confirmation requirements are explicit.

**Artifact criteria**
- Review checklist completed (not just started — every item addressed)
- Deployment runbook executed step by step (not skipped because "we've done
  this before")
- Rollback plan exists at a defined location before deployment begins

**Quality criteria**
- No open critical or blocking issues remain unresolved
- Rollback path has been tested or, if untestable in advance, its steps have
  been reviewed by someone other than the person who wrote them
- Performance or load characteristics verified against thresholds defined in
  SCOPE.md, if applicable

**Decision criteria**
- Go/no-go decision recorded explicitly, with the deciding party identified —
  not implied by the fact that deployment happened
- Any issues discovered during review that were accepted rather than resolved
  are recorded with rationale and risk acknowledgment

**Confirmation criteria**
- Explicit sign-off obtained before any irreversible action (production deploy,
  data migration, external notification)
- "Implicit confirmation" (no objections within N hours) is only acceptable
  if that confirmation model was defined in the chain before the phase began

---

## Anti-patterns in completion criteria

### 1. Criteria by Effort

"We spent three days on this, so it's done."

Effort is not a completion criterion. A phase that consumed the allocated time
may still be incomplete if the outputs do not meet the defined criteria. Effort
spent is a cost, not evidence of completion. Replace effort-based criteria with
output-based criteria: what exists, what passes, what was decided.

### 2. Vague Quality

"It looks good." "The code is clean." "Tests seem fine."

These are not measurable. A quality criterion must specify a signal that can
be checked without judgment: a score, a pass/fail result, a threshold. If the
only available signal is judgment, name the judge explicitly and specify what
they are evaluating against. "Senior engineer reviewed against SCOPE.md
requirements" is checkable. "Looks good" is not.

### 3. Moving Goalposts

Defining or revising completion criteria after the phase has started — usually
to match what the phase actually produced.

Criteria must be defined before the phase begins. If criteria prove wrong or
inapplicable during execution, update the chain definition explicitly and
record the change with rationale. Do not silently adjust criteria to match
output. The gate exists to catch mismatches between intent and output, not to
ratify whatever was produced.

### 4. Criteria Washing

Marking a phase CLEARED when criteria are partially met, to avoid the delay
of being BLOCKED.

Every criterion must be met for CLEARED. Partial credit does not exist.
If a criterion is consistently met except in one area, examine whether the
criterion is correctly defined or whether the phase plan is insufficient.
Criteria washing produces false confidence and loads debt onto later phases.

### 5. Missing Artifact Path

Criteria reference files or outputs that do not have a defined location.

"Test results exist" is incomplete if there is no defined path or format.
Every artifact criterion must specify where the artifact lives and in what
form. Artifact criteria without paths cannot be checked consistently across
context resets or agent handoffs.

---

## Completion criteria template

Use this template when defining a prereq-chain for a new workflow. Fill in
all fields before the phase begins. Leave no field blank — if a criterion type
does not apply to a specific phase, write "N/A — [reason]" so the omission is
intentional and visible.

```
Phase: [name — verb-noun form, e.g. "Design", "Implement", "Test", "Deploy"]

Artifact criteria:
  - [file or output that must exist]
  - [path or location where it must exist]
  - [any format or completeness requirement]

Quality criteria:
  - [measurable standard that must be met]
  - [how it is measured — tool, score, threshold]
  - [baseline or reference point if applicable]

Decision criteria:
  - [decision that must be recorded]
  - [where it is recorded]
  - [required elements: what was decided, what was rejected, why]

Confirmation criteria:
  - [who must confirm]
  - [what counts as confirmation — message, sign-off, checklist]
  - [where confirmation is recorded]

BLOCKED signal:
  Emit when any of the above criteria are unmet:
  "PHASE [N] → [N+1] GATE: BLOCKED
   [criterion type]: [specific unmet condition]
   Required: [what must happen before re-check]"
```

---

## Quick-reference: criteria sufficiency check

Before finalizing a phase's criteria, run this check:

- Can each criterion be verified without ambiguity by someone who did not
  do the work? If no, tighten the criterion.
- Does each criterion have a specific signal (file exists, score above X,
  decision recorded in Y)? If no, add the signal.
- Were all criteria defined before the phase began? If no, record the
  criteria change with rationale.
- Is there at least one artifact criterion and one quality or decision
  criterion per phase? If no, add the missing type.
- Does the final phase's confirmation criterion reference SCOPE.md success
  criteria? If no, add that reference.
