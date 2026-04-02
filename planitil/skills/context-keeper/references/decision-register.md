# Decision Register — Reference Guide

## What Is a Significant Decision

Apply the reversibility test: would undoing this choice cost more than two hours of work? If yes, record it in DECISIONS.md.

### Qualifies for Recording

- Choice of data store or persistence mechanism
- Choice of authentication or authorization strategy
- Choice to scope a requirement out or defer it to a later milestone
- Choice of deployment target (self-hosted, cloud provider, edge)
- Choice of inter-service communication protocol
- Choice of external dependency (library, vendor, API)
- Choice of testing approach or coverage target
- Any choice explicitly described as an "ADR" or "architectural decision"

### Does Not Qualify

- Variable naming or code style choices covered by a linter
- Minor UI layout adjustments
- Commit message wording
- Temporary choices flagged as "we will revisit before v1"

When uncertain, record it. The cost of an unnecessary entry is one minute. The cost of a missing entry is re-litigating a decision the team already made.

## Entry Format

```
## DEC-NNN: [Decision Title]

- **Date:** YYYY-MM-DD
- **What was decided:** [One sentence stating the decision clearly — present tense]
- **Why (rationale):** [Two to four sentences covering the problem, the choice, and the forcing function]
- **Alternatives considered:**
  - [Alternative 1] — [Decisive disqualifier]
  - [Alternative 2] — [Decisive disqualifier]
- **Status:** Active | Superseded | Revisited
```

IDs are sequential per project. Assign the next available NNN by reading the file and finding the highest existing number. Do not reuse IDs.

## Writing Rationale: The What-Why-Why-Not Pattern

Strong rationale answers three questions in sequence:

**What:** State the choice made, not the problem. Begin with the decision, not with the context. Weak: "We needed a fast data store, so..." Strong: "We use SQLite for all local storage."

**Why:** State the forcing function or constraint that made this choice correct — not a preference, a constraint. Weak: "We prefer SQLite." Strong: "The offline-first constraint in PROJECT.md eliminates any solution requiring network access at query time."

**Why not alternatives:** For each alternative considered, name the one decisive disqualifier. Not "it wasn't the best fit" — the specific reason this option was eliminated. Weak: "JSON was too slow." Strong: "Flat JSON requires full-file parse for every query; at 6 months of daily entries (≈ 10,000 rows), this exceeds the 50 ms response target defined in SPEC.md."

The why-not-alternatives section is the most valuable part of the record. When someone proposes switching to JSON in six months, this section answers them without requiring the original decision-maker to be in the room.

## Handling Superseded Decisions

When a decision is reversed or replaced:

1. Append a new entry (DEC-NNN+1) that references the original: "Supersedes DEC-NNN."
2. In the new entry's rationale, explain what changed — the constraint that was removed, the new information that arrived, or the assumption that was invalidated.
3. Update only the `Status` field of the original entry to: `Superseded — see DEC-NNN+1`.
4. Do not delete, rewrite, or reorganize the original entry.

The history of why a decision changed is as valuable as the decision itself. A team that can trace "we switched from X to Y in March because assumption ASS-003 was invalidated" has institutional memory. A team that deleted the original entry does not.

## Cross-Register Links

Add these notes to a DECISIONS.md entry when applicable:

- `Related risks: RISK-NNN` — when the decision creates or is associated with a logged risk
- `Closes: RISK-NNN` — when the decision resolves a previously open risk
- `Caused by: ASS-NNN being invalidated` — when the decision revises an earlier choice because an assumption failed validation
- `Creates: DEBT-NNN` — when the decision deliberately defers something

## File Initialization

If DECISIONS.md does not exist, create it with this header before the first entry:

```
# Decision Register

Tracks significant architectural and scope decisions. Append only — do not edit existing entries except to update Status fields.

---
```
