---
name: context-keeper
description: |
  Use when a significant decision, assumption, risk, or deferred item needs to be recorded — or when resuming work after a session reset and project context needs to be restored from persistent registers.

  <example>
  User: "Record this decision" / "Log this assumption" / "Add this to the risk register"
  Action: Run context-keeper to append a structured entry to the appropriate register file (DECISIONS.md, ASSUMPTIONS.md, RISKS.md, or DEBT.md) with date, rationale, and status.
  </example>

  <example>
  User has just confirmed an ADR or made an architectural choice during arch-design
  Action: Invoke context-keeper automatically to record the decision in DECISIONS.md, flag any new risks in RISKS.md, and capture any assumptions the decision relies on in ASSUMPTIONS.md.
  </example>
version: 1.0.0
---

# context-keeper

Maintain four living registers that survive session resets: DECISIONS.md, ASSUMPTIONS.md, RISKS.md, and DEBT.md. Every significant decision, assumption, risk, and deferred item is recorded with date, rationale, and status. These files are the project's institutional memory — they restore decision context without re-deriving it.

## What This Skill Maintains

Four files, kept in `.planning/` at the project root:

| File | Tracks |
|---|---|
| `DECISIONS.md` | Significant architectural and scope decisions |
| `ASSUMPTIONS.md` | Explicit and implicit assumptions being carried forward |
| `RISKS.md` | Identified risks with likelihood, impact, and mitigation |
| `DEBT.md` | Deferred work with a required reason and target date |

## DECISIONS.md

### Entry Format

```
## DEC-NNN: [Decision Title]

- **Date:** YYYY-MM-DD
- **What was decided:** [One sentence stating the decision clearly]
- **Why (rationale):** [Two to four sentences: the problem being solved, why this option, why the alternatives were not chosen]
- **Alternatives considered:** [Bulleted list of rejected alternatives, each with one-line rejection reason]
- **Status:** Active | Superseded | Revisited
```

### What Qualifies as a Significant Decision

Apply the reversibility test: would undoing this choice cost more than two hours of work? If yes, record it. Examples that qualify: choice of data store, choice of authentication strategy, choice to scope something out, choice of deployment target, choice of communication protocol, choice to defer a requirement.

Examples that do not qualify: variable naming, minor UI layout choices, commit message wording.

### Handling Superseded Decisions

When a decision is reversed or replaced, append a new entry (DEC-NNN+1) referencing the original. Update only the `Status` field of the original entry to `Superseded — see DEC-NNN+1`. Never delete or rewrite previous entries — the history of why a decision changed is as valuable as the decision itself.

### Rationale Pattern

Write rationale as "what, why, why-not-alternatives":

- **What:** State the choice made, not the problem.
- **Why:** State the forcing function or constraint that made this choice correct.
- **Why not alternatives:** For each alternative, name the decisive disqualifier — not a general preference.

## ASSUMPTIONS.md

### Entry Format

```
## ASS-NNN: [Assumption Statement]

- **Date made:** YYYY-MM-DD
- **Impact if wrong:** High | Medium | Low
- **Validation method:** [How this assumption will be confirmed — a specific test, spike, user interview, or measurement]
- **Status:** Unvalidated | Confirmed | Invalidated
```

### What Qualifies as an Assumption

An assumption is anything the project treats as true without confirmed evidence. Use the "if this is wrong, what breaks?" test:

- **High impact:** Wrong assumption requires restarting the affected phase or redesigning the architecture.
- **Medium impact:** Wrong assumption requires significant rework of a component.
- **Low impact:** Wrong assumption requires a minor adjustment or a small refactor.

Record both explicit assumptions (things the team knowingly accepted as true) and implicit assumptions (things no one questioned but that could be false). Implicit assumptions are more dangerous and more important to surface.

### Validation Methods by Assumption Type

- **Technical feasibility assumptions:** A time-boxed spike (timebox, hypothesis, pass/fail criteria).
- **User behavior assumptions:** User interview, usability test, or beta observation.
- **External dependency assumptions:** Proof-of-concept integration, API contract review, or vendor confirmation.
- **Regulatory or compliance assumptions:** Legal review or compliance team sign-off.

### Integration with confidence-gate

Assumptions identified during confidence-gate analysis are automatically appended to ASSUMPTIONS.md. The confidence-gate skill sets Status to `Unvalidated` and assigns impact based on the confidence score delta. Do not duplicate an entry already added by confidence-gate — check for an existing ASS-NNN entry on the same topic before appending.

## RISKS.md

### Entry Format

```
## RISK-NNN: [Risk Description]

- **Date identified:** YYYY-MM-DD
- **Likelihood:** High | Medium | Low
- **Impact:** High | Medium | Low
- **Mitigation strategy:** [Specific action to reduce likelihood or impact — not "monitor closely"]
- **Status:** Open | Mitigated | Accepted | Closed
```

### Likelihood/Impact Matrix and Response Strategies

| Likelihood | Impact | Default Strategy |
|---|---|---|
| High | High | Mitigate immediately — block progress until mitigation is in place |
| High | Low | Accept with monitoring trigger defined |
| Low | High | Mitigate proactively — add to DEBT.md if mitigation deferred |
| Low | Low | Accept — log and move on |

Response strategy definitions:

- **Mitigate:** Take a specific action that reduces likelihood or impact before it becomes a problem.
- **Accept:** Acknowledge the risk and proceed, with a defined trigger that would escalate to mitigation.
- **Transfer:** Assign the risk to a third party (vendor SLA, insurance, external team).
- **Avoid:** Change the approach to eliminate the risk entirely.

### Closing a Risk

Update Status to `Closed` only with evidence. Acceptable evidence: the risky component shipped successfully, the external dependency was confirmed, the spike resolved the uncertainty. "It didn't happen" is not evidence.

## DEBT.md

### Entry Format

```
## DEBT-NNN: [What Was Deferred]

- **Date deferred:** YYYY-MM-DD
- **Why deferred:** [Acceptable reason — see below]
- **Impact of deferral:** [What breaks or degrades if this is never resolved]
- **Target resolution date:** YYYY-MM-DD
- **Status:** Open | In Progress | Resolved
```

### Acceptable Deferral Reasons

Debt entries require an acceptable reason. Acceptable reasons:

- **Intentional shortcut:** A simpler approach chosen consciously to meet a deadline, with the known cost documented.
- **External constraint:** A dependency, vendor limitation, or regulatory requirement outside the team's control.
- **Sequencing constraint:** This work can only be done after another component exists.

Unacceptable reasons (never log debt with these rationales):

- "We'll do it later" without a target date.
- "It's too hard right now" without a specific blocker.
- "Not a priority" — this is not a reason, it is a consequence of never recording the debt.

### Debt That Is Never Acceptable

Do not log as debt: known security vulnerabilities with no mitigation, missing error handling on user-facing flows, removal of test coverage. These are defects, not debt — treat them as blocking issues.

### Target Dates Are Mandatory

A DEBT.md entry without a target resolution date is permanent debt. Every entry must have a date. If the date is genuinely unknown, write `Target: Before [milestone name]` — anchoring to a milestone is acceptable, anchoring to nothing is not.

## The Update Protocol

When invoked, follow this sequence:

1. **Identify which registers need updating.** A single invocation may update one, two, or all four files. A decision may create a new risk (RISKS.md) and surface a new assumption (ASSUMPTIONS.md) at the same time.

2. **Check existing entry IDs.** Read the target register file, find the highest existing NNN sequence number, and assign the next number. If the file does not exist, start at 001.

3. **Append the new entry.** Add the entry at the bottom of the file. Do not rewrite, restructure, or reorganize existing entries.

4. **Update Status fields only.** The only permitted edit to existing entries is updating the `Status` field (e.g., marking a decision Superseded, closing a risk, resolving debt). All other fields are append-only.

5. **Apply cross-register links.** If a DECISIONS.md entry creates a new risk, add a note in the decision entry: `Related risks: RISK-NNN`. If a DEBT.md entry was caused by a decision, note: `Caused by: DEC-NNN`.

## Cross-Register Linking

Registers are not isolated — link related entries across files:

- A DECISIONS.md entry should reference any RISKS.md entries it creates (`Related risks: RISK-NNN`) or closes (`Closes: RISK-NNN`).
- A DEBT.md entry should reference the decision or constraint that caused the deferral (`Caused by: DEC-NNN` or `External constraint: [description]`).
- An ASSUMPTIONS.md entry should reference any risk that depends on it (`Risk if invalidated: RISK-NNN`).

## Session Continuity

These four files survive `/clear` and session resets because they are written to disk. When resuming work after a reset:

1. Read DECISIONS.md to restore architectural context — what was decided and why.
2. Read ASSUMPTIONS.md to identify what the current plan depends on being true.
3. Read RISKS.md to understand what the team is monitoring or mitigating.
4. Read DEBT.md to know what was knowingly deferred and when it is due.

Loading these registers takes under five minutes and restores decision context that would otherwise require hours of conversation to re-derive.

## Writing the Registers

When appending or updating any register, write the updated file back to disk immediately at its canonical path under `.planning/`. Do not hold updates in memory — write after every invocation. Confirm with: `[REGISTER].md updated at .planning/[REGISTER].md`

## Worked Example

The team has just accepted an ADR: use SQLite for local storage instead of a flat JSON file.

**Step 1 — DECISIONS.md entry created:**

```
## DEC-001: Use SQLite for local time-entry storage

- **Date:** 2026-03-31
- **What was decided:** Store all time entries in a local SQLite database rather than a flat JSON file.
- **Why (rationale):** At 6+ months of daily use, a flat JSON file must be fully loaded and parsed for every query. SQLite supports indexed queries without full-file reads, and the Python standard library includes sqlite3 with no additional dependencies. The offline requirement is preserved since SQLite is file-based.
- **Alternatives considered:**
  - Flat JSON file — rejected: O(n) query cost at scale; no concurrent-write safety
  - Remote API / cloud sync — rejected: violates the offline-first constraint defined in PROJECT.md
- **Status:** Active
  Related risks: RISK-001
```

**Step 2 — RISKS.md entry created (linked from DEC-001):**

```
## RISK-001: SQLite schema migration complexity as feature set grows

- **Date identified:** 2026-03-31
- **Likelihood:** Medium
- **Impact:** Medium
- **Mitigation strategy:** Adopt Alembic for migration management from v0.1; write a migration script alongside every schema change PR; include a schema version check at CLI startup.
- **Status:** Open
```

**Step 3 — ASSUMPTIONS.md entry created:**

```
## ASS-001: Python stdlib sqlite3 module is sufficient; no ORM required

- **Date made:** 2026-03-31
- **Impact if wrong:** Medium — adopting an ORM (e.g., SQLAlchemy) mid-project would require rewriting all data-access code
- **Validation method:** Spike: implement the three core queries (insert entry, list today's entries, compute daily total) using raw sqlite3; pass/fail gate: all three queries complete in < 50 ms on a 10,000-row dataset
- **Status:** Unvalidated
  Risk if invalidated: RISK-001
```

## Common Mistakes

- **Updating entries instead of appending.** Editing a previous entry erases the history of how a decision evolved. Append only; update Status fields only.
- **Entries without dates.** An undated entry is unanchored in time — it cannot be correlated with what the team knew when the decision was made. Every entry requires `Date:`.
- **DEBT.md entries without a target date.** Debt without a due date is permanent. If the exact date is unknown, anchor to a milestone.
- **DECISIONS.md entries without the "why not alternatives" section.** The most valuable part of a decision record is why the alternatives were rejected. A decision entry that only states what was chosen provides no protection against re-litigating the same choice six months later.
- **Risk mitigation strategies that say "monitor closely."** Monitoring is not mitigation. A mitigation strategy must describe a specific action that reduces likelihood or impact.

## References

- `references/decision-register.md` — Full guide to DECISIONS.md: reversibility test, entry format, rationale pattern, handling superseded decisions
- `references/assumption-register.md` — Guide to ASSUMPTIONS.md: explicit vs implicit assumptions, impact assessment, validation methods by type
- `references/risk-debt-patterns.md` — Combined guide to RISKS.md and DEBT.md: likelihood/impact matrix, debt classification, closing entries with evidence
