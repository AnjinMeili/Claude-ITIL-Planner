# Assumption Register — Reference Guide

## What Is an Assumption

An assumption is anything the project treats as true without confirmed evidence. The test: "If this is wrong, what breaks?"

### Explicit vs Implicit Assumptions

**Explicit assumptions** are things the team knowingly accepted as true and stated out loud — "we assume users are on macOS or Linux." These are easy to capture because someone said them.

**Implicit assumptions** are things no one questioned because they seemed obvious — "we assume the sqlite3 stdlib module will handle our query volume." These are more dangerous because they are invisible until they fail. Surfacing implicit assumptions is the primary value of this register.

Signs of an implicit assumption:
- A choice that was made without discussion ("obviously we'll use X")
- A requirement that assumes a particular user behavior ("users will always...")
- A dependency on a third-party system without confirming its behavior
- A performance expectation with no measurement backing it

### What Qualifies for Recording

Record an assumption when:
- A plan or design depends on it being true
- Being wrong would require rework at Medium or High cost
- It has not been validated by a test, spike, interview, or confirmed document

### What Does Not Qualify

Do not record confirmed facts. "Python 3.11 supports match statements" is not an assumption — it is a fact verifiable in the changelog. "Our users are on Python 3.11 or later" is an assumption — it has not been confirmed for the actual user population.

## Impact Assessment

Impact measures the cost of invalidation — what would need to change if this assumption turned out to be wrong.

**High impact:** Wrong assumption requires restarting the affected phase or redesigning the architecture. Example: "We assumed users have write access to ~/.local/ — invalidated means we need to redesign the storage location strategy and re-implement the installer."

**Medium impact:** Wrong assumption requires significant rework of a component. Example: "We assumed the sqlite3 stdlib module handles 10,000-row queries in < 50 ms — invalidated means we need to add an index layer or adopt an ORM."

**Low impact:** Wrong assumption requires a minor adjustment or small refactor. Example: "We assumed users prefer 24-hour time format in the summary output — invalidated means we add a --format flag."

When uncertain between two levels, use the higher one. The cost of over-preparing is lower than the cost of an unmitigated invalidation.

## Validation Methods by Assumption Type

**Technical feasibility assumptions**
Validate with a time-boxed spike. Define: hypothesis, timebox (hours), and pass/fail criteria. A spike without pass/fail criteria is not a validation — it is exploration. Example: "Hypothesis: sqlite3 handles 10,000-row queries in < 50 ms. Timebox: 2 hours. Pass: all three core queries complete under target. Fail: any query exceeds target."

**User behavior assumptions**
Validate with user interviews, usability tests, or beta observation. Specify: number of participants, what behavior is being measured, and what result confirms or invalidates the assumption. Example: "Interview 5 developers. Ask them to recall yesterday's task distribution from memory. Pass: 4/5 cannot recall accurately — confirms the problem exists. Fail: 4/5 recall accurately — invalidates the problem statement."

**External dependency assumptions**
Validate with a proof-of-concept integration, API contract review, or vendor confirmation. Specify the exact claim being validated. Example: "PyPI publishing: publish a test package to TestPyPI and install it. Pass: install succeeds in a clean virtual environment. Fail: any error."

**Regulatory or compliance assumptions**
Validate with legal review or compliance team sign-off. Do not self-assess compliance. Example: "Assumption: storing time entries locally with no cloud sync requires no GDPR compliance actions. Validation: legal team review. Pass: written confirmation. Fail: any action items identified."

**Performance assumptions**
Validate with a benchmark. Specify: dataset size, hardware profile (or minimum spec), measurement method, and target. Benchmarks run on developer hardware are not valid for server or edge targets — match the environment to the deployment context.

## Entry Format

```
## ASS-NNN: [Assumption Statement]

- **Date made:** YYYY-MM-DD
- **Impact if wrong:** High | Medium | Low
- **Validation method:** [Specific test, spike, interview, or document that will confirm or invalidate this assumption]
- **Status:** Unvalidated | Confirmed | Invalidated
```

Write the assumption statement as a positive claim: "Users have Python 3.11 or later installed." Not: "We don't know which Python version users have."

## Status Transitions

- **Unvalidated → Confirmed:** Add a note below the Status line: `Confirmed by: [validation performed] on YYYY-MM-DD`.
- **Unvalidated → Invalidated:** Add a note: `Invalidated by: [what was discovered] on YYYY-MM-DD`. Then check which decisions or plans depend on this assumption and flag them for review.
- Never delete an Invalidated entry — the invalidation event is part of the project record.

## Integration with confidence-gate

The confidence-gate skill surfaces assumptions as part of its analysis. When confidence-gate runs, it appends Unvalidated entries to ASSUMPTIONS.md for any assumption it identifies with a confidence impact of 0.2 or higher. Before appending, check for an existing entry on the same topic to avoid duplication.

## File Initialization

If ASSUMPTIONS.md does not exist, create it with this header:

```
# Assumption Register

Tracks explicit and implicit assumptions the project is carrying forward. Append only — do not edit existing entries except to update Status fields.

---
```
