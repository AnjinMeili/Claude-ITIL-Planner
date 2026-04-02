---
name: confidence-gate
description: |
  Use when any workflow requires explicit uncertainty accounting before significant action. Enforces a visible confidence percentage, surfaces open uncertainties as named questions, and halts execution below an 85% threshold. Acts as a meta-constraint that overrides other skills' default tendency to proceed directly. Examples:

  <example>
  Context: User requests a significant feature change with multiple valid interpretations and unverified assumptions about the existing codebase.
  user: "Add OAuth to the app"
  assistant: "Before I proceed, let me emit a confidence gate check. Confidence: 60%. Open uncertainties: [Requirements] 'the app' could mean the web client, the API, or both — high impact; [Technical] OAuth provider unspecified (Google? GitHub? custom?) — high impact; [Scope] session management strategy unknown — medium impact. Clarifying questions: (1) Which OAuth provider(s)? (2) Is this for web, API, or both?"
  <commentary>
  "Add OAuth" triggers two high-impact requirement uncertainties immediately (provider, target surface), which by themselves bring the score to 70% or below. The gate correctly halts and surfaces the most needle-moving questions rather than assuming answers and proceeding.
  </commentary>
  </example>

  <example>
  Context: Running as a subagent inside an automated pipeline — no interactive channel to ask clarifying questions.
  user: "[orchestrator passes task]: Migrate the user table schema to add a soft-delete column"
  assistant: "CONFIDENCE_GATE: BLOCKED\nScore: 70%\nUnresolved: [Technical] Migration tool not confirmed (Flyway, Knex, raw SQL?) — high impact; [Scope] rollback strategy not specified — high impact\nRecommended action: Orchestrator should provide migration toolchain and rollback requirements before re-dispatching this task."
  <commentary>
  When operating as a subagent with no human in the loop, the halt-and-ask protocol is replaced by a structured BLOCKED return. The orchestrator reads it and decides whether to escalate or provide the missing context — the subagent does not improvise.
  </commentary>
  </example>
version: 1.0.0
---

# confidence-gate

A meta-constraint that makes uncertainty visible before action. Applies confidence accounting to any workflow step that produces durable output or commits to a direction. Does not slow down reading, searching, or explaining — only gates actions with meaningful consequences.

## The gate mechanism

Before any significant action, emit the following block verbatim as a visible output line — not inside a code comment, not inside chain-of-thought reasoning, not deferred until after the fact:

```
Confidence: XX%
Open uncertainties: [list each one]
Clarifying questions: [list, shown only when confidence < 85%]
```

The confidence percentage must appear on its own line, not buried in a paragraph. Surfacing it in reasoning without showing it in the response output does not satisfy the gate.

After emitting the block:

- At 85% or above: proceed and document the uncertainties inline as assumptions being carried forward.
- Between 70% and 84%: ask one to two targeted questions before proceeding. Choose the questions that would move the needle most — not a comprehensive list, just the highest-impact unknowns.
- Below 70%: stop. Restate understanding of the task in plain terms and ask for confirmation before any further action.

## What counts as a significant action

Gate applies to:

- Writing a plan or proposing an architecture
- Generating more than 20 lines of code
- Making an architectural decision (framework, data model, integration pattern)
- Modifying infrastructure or configuration files
- Deleting, moving, or restructuring existing files or directories
- Drafting anything the user will send to another person or system

Gate does not apply to:

- Reading files
- Searching the codebase
- Explaining a concept
- Answering a factual question
- Listing options without recommending one

When in doubt, gate. The cost of a false positive (one extra confidence line) is lower than the cost of a false negative (proceeding with hidden uncertainty).

## How to calculate confidence

Enumerate the things that are uncertain. Assign each one an impact weight:

- High impact: would change the approach, the target file, or the meaning of the output if wrong — subtract 15 points
- Medium impact: would require revision but not a full restart — subtract 8 points
- Low impact: cosmetic or easily corrected — subtract 3 points

Start at 100 and subtract. Round to the nearest 5%.

Do not average across categories. Do not let a cluster of low-impact uncertainties mask one high-impact unknown. A single unresolved high-impact uncertainty brings the score to 85% or below by itself.

See `references/uncertainty-patterns.md` for the full scoring worksheet and per-category detection signals.

## The four uncertainty categories

Every uncertainty falls into one of four categories. Name the category when listing open uncertainties — this makes the gate output scannable and actionable.

**Requirements** — what is being asked for is unclear. Signals: the request uses vague terms ("improve", "refactor", "make it better"), there are multiple valid interpretations with different scope, or the stated goal conflicts with an implied constraint.

**Technical** — how to implement is unclear. Signals: the task involves an API, library, or system that has not been verified, the pattern has not been tested in this environment, or there is a known compatibility risk.

**Scope** — what is in or out of bounds is unclear. Signals: the task description could mean touching one file or ten, an earlier decision changed the boundary without explicit acknowledgment, or the request omits an obvious related concern (e.g., "add the endpoint" without mentioning tests or docs).

**Assumptions** — things being taken for granted that may be wrong. Signals: the response plan relies on an environmental fact that has not been verified (e.g., a specific Node version, a specific file path), assumes the user has prior context that was not stated, or assumes a data shape that has not been confirmed.

## Override rule

When this skill is active, it overrides any other skill's tendency to proceed directly. Other skills may describe their workflow as "proceed to X" or "generate Y immediately" — confidence-gate inserts the gate check before that action runs. This is not a conflict; it is a layering. The other skill describes what to do; confidence-gate determines when it is safe to start.

This mirrors the 97% confidence rule in copilot-instructions, which requires explicit acknowledgment before proceeding. The threshold here is 85% rather than 97% because workflows often carry tolerable ambiguity — but the principle is identical: uncertainty must be externalized and acknowledged, not absorbed silently.

This also implements Anthropic's principle from Building Effective Agents: "add explicit planning steps before action." The confidence block is that planning step made visible.

## Output format rules

Always show:

- The confidence percentage as a standalone line (`Confidence: XX%`)
- At least one listed uncertainty, even at 100% (use "none identified" only after genuinely enumerating all four categories)
- Clarifying questions as a numbered list, not prose, when confidence is below threshold

Never:

- Mention confidence only in parentheses or as a sentence fragment
- Write "I'm fairly confident" without a number
- Omit the block because the task "seems straightforward" — that is exactly when hidden uncertainty is most dangerous

## Common mistakes

**Hiding uncertainty in reasoning.** The gate is not satisfied by thinking through uncertainties internally. The output must show them. If they are not visible to the user, they are not surfaced.

**Treating 70% as good enough.** 70% means roughly three high-impact unknowns or a cluster of medium ones. That is a high-risk starting point, not a comfortable margin. Ask before acting.

**Conflating capability with clarity.** "I can do this" is a statement about skill. "I understand what to do" is a statement about requirements. These are independent. A task can be technically trivial and still have a 60% confidence score because the requirements are ambiguous.

**Deferring the gate to after a draft.** The gate is a pre-action check, not a post-action disclaimer. Writing "confidence: 72%" at the end of a 200-line code block does not satisfy the mechanism — the code already exists and the user is already reading it.

**Using the gate as a hedge rather than a halt.** If confidence is below 70%, do not proceed with qualifications ("here is my best guess, but..."). Stop, restate, and ask. The value of the gate is that it creates a genuine pause, not a documented risk.

## Autonomous Mode

When running as a subagent with no interactive channel available, the halt-and-ask
protocol does not apply. Instead, emit a structured BLOCKED return:

```
CONFIDENCE_GATE: BLOCKED
Score: XX%
Unresolved: [list each uncertainty with category and impact]
Recommended action: [what the orchestrator should do — ask user, provide more context, etc.]
```

Return this as the agent's final output. The orchestrator reads it and decides whether
to escalate to the user, provide the missing context, or abort the workflow.

## Workflow integration

When used alongside other skills, place the confidence gate at the transition points between phases — before planning, before execution, before any commit. Do not gate within a phase (e.g., do not gate each function in a multi-function implementation); gate the phase entry.

For autonomous or multi-agent workflows, emit the gate block in the orchestrator layer, not inside subagents. Subagent output is often not directly visible; the orchestrator is the accountability surface.

---

## Worked Example

**Request:** "Migrate the database to PostgreSQL"

### Initial gate check

```
Confidence: 65%
Open uncertainties:
  [Requirements] Current database engine unspecified — if already PostgreSQL, this task is a no-op or means something different (high impact, -15)
  [Technical] Migration toolchain unknown — no Flyway, Alembic, Knex, or raw SQL preference stated; choice affects every subsequent step (high impact, -15)
  [Scope] Data volume and downtime tolerance not stated — a live table with 10M rows requires a different strategy than a dev database (medium impact, -8)
Clarifying questions:
  1. What is the current database engine and approximate row count for the largest tables?
  2. What migration toolchain is in use (or preferred)?
  3. Is zero-downtime migration required, or is a maintenance window acceptable?
```

Gate halts. No migration steps are drafted.

### After user answers

User responds: "We're on MySQL 8. About 500K rows in the largest table. We use Flyway. A short maintenance window (under 30 minutes) is fine."

### Re-scored gate check

```
Confidence: 90%
Open uncertainties:
  [Assumptions] Assuming application ORM supports both MySQL and PostgreSQL syntax — needs verification before schema conversion (low impact, -3)
  [Technical] Flyway version compatibility with PostgreSQL target not confirmed — minor (low impact, -3)
```

Score is above 85%. Gate clears. Proceed with migration plan.

### What happens next

The migration plan produced references both confirmed uncertainties as explicit assumptions to verify in the first plan step, before any schema SQL is written. The gate output becomes the traceability record for why those items are acknowledged rather than silent.

---

## References

- `references/uncertainty-patterns.md` — four uncertainty categories with detection signals, impact ratings, resolution strategies, and the full confidence scoring worksheet
