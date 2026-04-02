# Acceptance Criteria Guide

Guide for writing well-formed Given/When/Then acceptance criteria in SPEC.md. Covers the format, the testability test, how criteria map to test types, and five untestable-to-testable rewrites.

---

## The Given/When/Then Format

Every acceptance criterion uses three clauses:

### Given — Precondition

Describes the state of the world before the triggering action occurs. Includes:
- System state (what data exists, what mode the system is in)
- User state (who is acting, what they have already done)
- Environmental state (what external systems are available or unavailable)

The Given clause must be specific enough that a test can be set up reproducibly. Vague Given clauses produce tests that sometimes pass and sometimes fail depending on incidental state.

Vague: "Given I am logged in..."
Specific: "Given an authenticated session exists with a valid Jira API token that expires in 60 minutes..."

### When — Triggering Action

Describes the single action that triggers the behavior under test. One action per When clause. Compound When clauses (When I do X and then Y) signal a test that covers too much ground — split it.

The When clause is at the interface level, not the implementation level. It describes what the user or external system does, not what the code does internally.

Interface level: "When I run `track start PROJ-123`..."
Implementation level: "When the CLI calls `createEntry()` with task ID PROJ-123..." — this is not an acceptance criterion, it is a unit test assertion. Keep it in the test code.

### Then — Observable Outcome

Describes the verifiable state of the world after the action. Every Then clause must pass the testability test (below).

Then clauses describe observable outcomes, not internal state. "Then the entry is stored in the database" requires access to database internals to verify. "Then running `track list` shows an entry with task PROJ-123 and the current timestamp" is observable without internal access.

---

## The Testability Test

Apply this test to every Then clause before accepting it:

**Question 1: Can this be confirmed as true or false without human judgment?**

If the answer is "it depends on interpretation" or "someone needs to decide if this counts," the criterion is not testable. Rewrite it.

Not testable: "Then the CLI outputs a helpful error message."
Testable: "Then the CLI outputs 'Authentication required. Run `track auth` to connect your Jira account.' and exits with code 1."

**Question 2: Can this be confirmed without reading source code?**

Acceptance criteria verify behavior, not implementation. If confirming the Then clause requires inspecting variables, database rows, or log files that a user would not access, rewrite it to describe the observable behavior.

Requires source access: "Then the `timerActive` flag is set to true in the application state."
Observable: "Then running `track status` outputs 'Timer running: PROJ-123 (started 14:32)'."

**Question 3: Can this be confirmed repeatably?**

If the test result depends on timing, network conditions, or incidental state not captured in the Given clause, the criterion is not reliable. Add the missing conditions to Given, or rewrite the criterion to remove the dependency.

Non-repeatable: "Then the sync completes quickly."
Repeatable: "Then the sync completes within 5 seconds when the Jira API responds within its documented 2-second SLA."

---

## Mapping Criteria to Test Types

Label each acceptance criterion with the test type needed to verify it. This label appears in SPEC.md and is used by QA.md to assign test implementation.

### Unit

The Then clause is verifiable with no external system or infrastructure. The test runs against a single component or function in isolation.

Signals:
- Given clause contains only local state or in-memory fixtures
- Then clause is verifiable without file system, network, or database
- Test runs in milliseconds

Example AC labeled Unit:
> AC-01-01 (Unit): Given no prior entries exist, when I run `track start PROJ-123`, then a timer entry object is returned with `taskId: "PROJ-123"` and a `startTime` within 1 second of the current system clock.

### Integration

The Then clause requires two or more components working together, or requires interaction with an external system, real file system, or real database.

Signals:
- Given clause involves data written to disk or a database
- Then clause requires verifying state in a second component
- Test may require setup/teardown of infrastructure (test DB, mock server)

Example AC labeled Integration:
> AC-02-01 (Integration): Given a valid Jira API token is configured and one unsynced local entry exists for PROJ-123, when I run `track sync`, then the Jira ticket PROJ-123 shows a work log entry with the correct duration and timestamp within 10 seconds.

### End-to-End

The Then clause requires a full user journey from initial input to a final observable outcome, typically spanning multiple use cases or system boundaries.

Signals:
- Given clause involves a user who has completed prior steps (authentication, configuration)
- Then clause verifies a state visible to an external system or a different user
- Test replicates a real usage scenario, not a synthetic one

Example AC labeled End-to-End:
> AC-03-01 (E2E): Given a new user has installed the CLI and has a valid Jira account, when the user completes the `track auth` flow and runs `track start PROJ-123` followed by `track sync`, then the Jira ticket PROJ-123 shows a time log entry and `track list` shows the entry as synced — all within a single terminal session.

---

## Untestable-to-Testable Rewrites

### Rewrite 1

**Untestable:** "Then the user receives a helpful error message when authentication fails."

**Problem:** "Helpful" is a judgment call. No automated test can assert "helpful."

**Testable:** "Then the CLI outputs exactly: 'Authentication failed. Your Jira token may be expired. Run `track auth --refresh` to update it.' and exits with code 2."

---

### Rewrite 2

**Untestable:** "Then the time entry is saved correctly."

**Problem:** "Saved correctly" is undefined. What does correct mean? Where is it saved? What fields must be present?

**Testable:** "Then running `track list --today` outputs one entry with task ID matching the argument, elapsed time between 0 and 5 seconds, and status 'active'."

---

### Rewrite 3

**Untestable:** "Then the sync is fast."

**Problem:** "Fast" is relative and unmeasurable as written. Fast compared to what? Under what network conditions?

**Testable:** "Then the sync command completes and exits within 8 seconds when processing 50 or fewer entries and the Jira API responds within its documented 2-second p95 latency."

---

### Rewrite 4

**Untestable:** "Then the system handles the error gracefully."

**Problem:** "Gracefully" requires human judgment. There is no automated assertion for gracefulness.

**Testable:** "Then the CLI outputs an error message beginning with 'Jira sync failed:', logs the error with timestamp to `~/.track/errors.log`, exits with code 3, and leaves all local entries in their pre-sync state."

---

### Rewrite 5

**Untestable:** "Then the data is not lost."

**Problem:** This is too broad. What data? Lost how? Under what failure conditions?

**Testable:** "Given a timer is running and the CLI process is killed with SIGTERM, then running `track list` after restart shows the interrupted entry with its start time preserved and status 'interrupted', with no entries missing from before the kill signal."

---

## Common Criteria Mistakes

**Criteria that describe internal implementation.** Then clauses that reference functions, variables, database tables, or log levels are asserting implementation rather than behavior. Move these assertions into the test code itself and rewrite the acceptance criterion to describe the observable output.

**Given clauses that leave preconditions ambiguous.** "Given I am logged in" could mean many things. Is the token about to expire? Is the network available? Is the local database empty? Ambiguous Given clauses produce tests that fail intermittently without a clear cause.

**Missing failure path criteria.** Every use case needs at least one failure path criterion — what happens when the expected preconditions are not met. A use case with only happy-path criteria will produce a system with undefined behavior on errors.

**Then clauses with compound assertions.** "Then the entry is saved and the sync succeeds and the terminal shows a confirmation" is three assertions in one criterion. Split into three criteria. Compound assertions make it impossible to identify which behavior failed when a test fails.

**Criteria that duplicate other criteria.** When two use cases share an actor and context, their acceptance criteria sometimes overlap. Duplicated criteria create maintenance burden — when behavior changes, two criteria must be updated. Consolidate into the use case where the behavior originates and reference by ID in the other.
