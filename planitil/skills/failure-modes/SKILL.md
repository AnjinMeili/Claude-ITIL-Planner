---
name: failure-modes
description: |
  Use when about to execute a high-stakes task — planning, architecture, or
  implementation — to surface cataloged failure modes before work begins. Generates
  a task-type-specific checklist of named failure patterns with explicit STOP
  conditions that signal "halt and restart." Makes predictable failure reasons
  visible upfront rather than discovered mid-execution. Examples:

  <example>
  Context: User signals they are about to write a plan for a significant feature area.
  user: "I'm about to write the plan for the auth system"
  assistant: "I'll invoke failure-modes before we start planning. Let me load the planning failure mode catalog and surface the checklist so we can confirm none of these apply before committing to a plan structure."
  <commentary>
  "About to write the plan" is the canonical planning trigger. The user has explicitly flagged the upcoming task type, making this a clean classification — run the planning catalog before any plan content is produced.
  </commentary>
  </example>

  <example>
  Context: Team is pausing before implementation to review the architecture they've sketched.
  user: "Let's review the architecture before we start coding"
  assistant: "Good moment to run failure-modes. I'll pull the architecture checklist — these are the named failure patterns most likely to surface when we review the structural decisions. Once we've confirmed none apply, we can proceed to code with confidence."
  <commentary>
  Architecture review maps directly to the architecture task type in the catalog. The phrase "before we start coding" confirms this is a pre-execution check, which is exactly when failure-modes should run.
  </commentary>
  </example>
version: 1.0.0
---

# failure-modes

A pre-execution checklist generator. Before starting any plan, architectural decision, or feature implementation, identify the task type, load the relevant failure mode catalog, present the checklist, and require acknowledgment before proceeding.

Most AI-assisted workflows fail for predictable, cataloged reasons. Surfacing those reasons before execution is cheaper than discovering them at step 7 of 10.

---

## When to invoke

Invoke before:

- Writing any plan or work breakdown
- Making any architectural decision (technology choice, service boundary, data model)
- Generating implementation code for a feature or module

Do not invoke for:

- Bug fixes with a clear reproduction case
- Exploratory research or spike work
- Single-file edits with well-defined scope
- Tasks where a `SCOPE.md` has already been confirmed and acknowledged in the current session

### Sequencing with scope-anchor

Run `scope-anchor` first, then `failure-modes`. Failure modes are most useful when scope is already defined — the catalog references `SCOPE.md` at multiple points. If scope is undefined, the most important failure mode is already active (Scope Inflation, Scope Escape), but the checklist will be noisy.

---

## Protocol

### Step 1 — Identify task type

Classify the task into exactly one of three types:

| Type | Trigger |
|---|---|
| **planning** | Producing a plan, work breakdown, sequence of steps, or roadmap |
| **architecture** | Deciding on technology, service boundaries, data models, or system structure |
| **implementation** | Writing or generating code for a feature, module, or integration |

If the task spans multiple types (e.g., a plan that includes architectural decisions), run the checklist for each applicable type.

### Step 2 — Load the catalog

Load the reference document for the identified type:

- Planning: `references/failure-modes/planning.md`
- Architecture: `references/failure-modes/architecture.md`
- Implementation: `references/failure-modes/implementation.md`

### Step 3 — Present the checklist

Present all failure modes from the relevant catalog as a numbered checklist. Each item follows this format:

```
N. FLAG NAME
   Symptom: [what this looks like when it's happening]
   STOP condition: [the specific observable signal that means halt and restart]
```

State the STOP condition precisely. A STOP condition is not a risk or a concern — it is a specific, observable signal. If the signal is present, do not continue. Restart with the condition addressed.

### Step 4 — Require acknowledgment

After presenting the checklist, ask:

> **Do any of these apply to the current task?**

Wait for a response. Do not proceed until the response is received.

- If **none apply**: confirm and proceed
- If **one or more apply**: address each flagged item before proceeding — do not work around them, do not continue and "note the risk"
- If **uncertain on any item**: treat it as applying and address it

---

## Output format

The checklist output:

```
## Failure Mode Check — [Task Type]

Before proceeding, review the following failure modes for this task type.

1. FLAG NAME
   Symptom: ...
   STOP condition: ...

2. FLAG NAME
   Symptom: ...
   STOP condition: ...

[... all items for the task type ...]

---

Do any of these apply to the current task?
```

Keep the checklist compact. The symptom description is one sentence. The STOP condition is one sentence stating the observable signal. Do not editorialize or add commentary — the format is a checklist, not an essay.

---

## STOP condition semantics

A STOP condition has a specific meaning in this protocol:

- It is **observable** — a person reviewing the work product can determine whether the signal is present without judgment calls
- It is **binary** — present or not present, not "partially present" or "a little bit"
- It **means halt** — if the signal is present, stop execution and restart with the condition resolved, not addressed in a comment, not noted as a known issue, not deferred to a later step

Examples of valid STOP signals:

- "any plan step that says 'as discussed' or 'per the requirements'" — observable, binary
- "any function named `get*` that has side effects" — observable, binary

Examples of invalid STOP signals (too vague):

- "when the plan seems underspecified" — not binary
- "when there is risk" — not observable

---

## Failure mode catalog summary

The three reference documents together cover 24 named failure modes across the three task types. Each failure mode has a name, a symptom pattern, and a STOP condition.

| Type | Count | Reference |
|---|---|---|
| planning | 8 | `references/failure-modes/planning.md` |
| architecture | 8 | `references/failure-modes/architecture.md` |
| implementation | 8 | `references/failure-modes/implementation.md` |

---

## Design rationale

This skill encodes two patterns:

**Defensive anti-pattern writing** — the failure modes are named anti-patterns, not vague warnings. Naming a failure mode (Orphaned Goal, Coupling Camouflage, Error Swallowing) makes it recognizable mid-execution. A named pattern is easier to flag than a general concern.

**Evaluator pattern** — the checklist is not a self-assessment. It is a structured external check applied before work starts. The model generating the plan should not also be the sole judge of whether the plan has these failure modes — the checklist makes the criteria explicit so they can be applied consistently.

---

## Worked Example

**Request:** "I'm about to write the implementation plan for the payment processing integration"

### Step 1 — Classify task type

Task type: **planning** (producing a plan for a module implementation)

### Step 2 — Load catalog

Loading `references/failure-modes/planning.md` — 8 named planning failure modes.

### Step 3 — Present checklist

```
## Failure Mode Check — Planning

Before proceeding, review the following failure modes for planning tasks.

1. SCOPE INFLATION
   Symptom: The plan includes tasks that were not in the original request or SCOPE.md.
   STOP condition: Any plan task that cannot be traced to a line item in SCOPE.md.

2. ORPHANED GOAL  [APPLIES]
   Symptom: Plan tasks exist that serve a different goal than the one stated — often
   imported from a related project or a previous version of the request.
   STOP condition: Any plan task where removing it would not reduce the plan's ability
   to achieve the stated goal.

3. SINGLE-STEP COMPRESSION
   Symptom: A complex phase is collapsed into one step ("implement payment flow") with
   no sub-steps.
   STOP condition: Any plan step that contains the words "implement", "build", or "create"
   without at least three sub-steps or a reference to a lower-level breakdown.

4. MISSING ROLLBACK  [APPLIES]
   Symptom: The plan includes irreversible operations (external API registration,
   database migrations, stored credentials) with no corresponding undo path.
   STOP condition: Any irreversible operation in the plan that has no rollback step
   or mitigation immediately following it.

5. SUCCESS CRITERIA OMISSION
   Symptom: The plan has no verifiable completion condition — no tests, no review gate,
   no observable signal that the work is done.
   STOP condition: The plan's final step is an action ("deploy", "merge") rather than
   a verification ("confirm X passes", "review Y against SCOPE.md").

6. IMPLICIT DEPENDENCY
   Symptom: A plan step assumes an artifact, decision, or environment that is not
   produced by a prior step and is not already confirmed to exist.
   STOP condition: Any plan step whose inputs are not produced by a prior step or
   confirmed present in the environment.

7. PRIORITY INVERSION
   Symptom: High-risk or foundational steps are scheduled late in the plan, after
   dependent work has already been done.
   STOP condition: Any step marked as foundational or irreversible that appears after
   steps that depend on its output.

8. SCOPE ESCAPE
   Symptom: The plan's total surface area visibly exceeds the SCOPE.md boundary —
   extra integrations, extra edge cases, extra phases added during planning.
   STOP condition: Any plan step that addresses a SCOPE.md out-of-scope item, even
   if it "only takes a minute."

---

Do any of these apply to the current task?
```

### Step 4 — Flags acknowledged; response received

User reviews and responds: "Yes — items 2 and 4 apply. We have a task in the draft plan for 'update admin dashboard' that crept in from a related project (Orphaned Goal). And we have a Stripe webhook registration step with no rollback."

### What happens next

Before producing any plan content:

1. **Orphaned Goal** — Remove the "update admin dashboard" task from the plan. If it is genuinely needed, open a new scope anchor session for it separately.
2. **Missing Rollback** — For the Stripe webhook registration step, add an immediately following step: "If registration fails or must be reverted, delete the webhook endpoint via Stripe Dashboard or API and remove the stored webhook secret from environment config."

Only after both flags are resolved does planning proceed. The checklist output is retained as a pre-plan artifact, referenced in the plan's header.

---

## References

- `references/failure-modes/planning.md` — 8 named planning failure modes with STOP conditions
- `references/failure-modes/architecture.md` — 8 named architecture failure modes with STOP conditions
- `references/failure-modes/implementation.md` — 8 named implementation failure modes with STOP conditions
