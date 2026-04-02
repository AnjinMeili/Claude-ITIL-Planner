---
name: qa-strategy
description: |
  Produces QA.md — the test strategy, coverage targets, UAT criteria, and observability specification. Defines how the system is verified before and after deployment.

  <example>
  User says "how should we test this" or "set up the QA plan" — invoke qa-strategy to produce QA.md covering the test matrix, UAT criteria, regression scenarios, and observability specification.
  </example>

  <example>
  After DEV-SPEC.md is confirmed, before test implementation begins — invoke qa-strategy to establish what must be tested, at what coverage threshold, and what constitutes a passing UAT.
  </example>
version: 1.0.0
---

# qa-strategy

## What This Skill Produces

Produce `QA.md` at `.planning/QA.md`. The file contains four sections: Test Matrix, UAT Criteria, Regression Scenarios, and Observability Specification. Every UAT criterion traces back to an acceptance criterion in SPEC.md. Every regression scenario names what could break it and why its failure would indicate something fundamental is wrong.

---

## Prerequisite Chain

Invoke qa-strategy after:
1. SPEC.md is confirmed (requirements-spec skill) — provides the acceptance criteria that UAT criteria are derived from
2. ARCH.md is confirmed (arch-design skill) — provides the components that populate the test matrix
3. DEV-SPEC.md is confirmed (dev-discipline skill) — provides the coverage thresholds and tooling choices

Do not produce QA.md without SPEC.md. UAT criteria written without traceability to acceptance criteria are untethered speculation.

---

## Section 1 — Test Matrix

Build a table with one row per logical component identified in ARCH.md. Every component has a test type, a tool, a coverage target, and an owner.

**Format:**

| Component | Type | Tool | Coverage Target | Owner |
|---|---|---|---|---|
| \<component name\> | unit / integration / E2E | \<tool name\> | \<percentage or criteria\> | \<team or agent\> |

**Column definitions:**

- **Component:** Name the component as it appears in ARCH.md. Do not rename components in the test matrix — trace breaks.
- **Type:** Select the highest-value test type for this component. Many components appear in multiple rows (one row per type). See `references/test-matrix-patterns.md` for how to choose.
- **Tool:** Name the specific tool or framework. "Unit tests" is not a tool. `pytest`, `jest`, `go test`, `playwright` are tools.
- **Coverage Target:** Express as a percentage for unit tests. Express as "all critical paths" for E2E. Do not set targets you will not enforce. If the project has no coverage measurement tooling, add it before setting percentage targets.
- **Owner:** The team, role, or agent responsible for maintaining the tests. Ownerless tests are abandoned tests.

**Coverage target guidance:**

- Unit tests: 80% line coverage on business logic modules is a realistic default. 100% is almost always the wrong target (it incentivizes trivial tests to hit the number).
- Integration tests: Cover the contract — does this component interact with its dependency correctly? Percentage coverage is the wrong metric; scenario coverage is right.
- E2E tests: Cover critical user-facing workflows only. E2E tests are expensive to maintain. More is not better.

---

## Section 2 — UAT Criteria

Derive every UAT criterion directly from SPEC.md acceptance criteria. Every acceptance criterion (AC-NN-MM) maps to at least one UAT scenario.

**Format per criterion:**

```
### UAT-[AC-NN-MM]: <short name>

**Source:** AC-NN-MM — <original acceptance criterion text>
**Given:** <the starting state a human tester can establish without code knowledge>
**Action:** <what the tester does — a user action, not a function call>
**Expected outcome:** <what the tester observes — in the UI, in a file, on the command line>
**Pass signal:** <the unambiguous indicator that this criterion is met>
```

**UAT criteria must be human-executable without code knowledge.** A UAT criterion that says "verify the `process_payment` function returns `{'status': 'ok'}`" is not a UAT criterion — it is a unit test. Translate it: "Submit a payment with a valid test card. Verify the confirmation screen appears with the order number."

**Traceability is mandatory.** Every UAT criterion references its AC-NN-MM source. If an acceptance criterion cannot be translated to a human-executable scenario, escalate to SPEC.md clarification — do not write an untraceable UAT criterion.

**When one AC maps to multiple UAT criteria:**

Some acceptance criteria cover a behavior with multiple observable outcomes or paths. Create one UAT criterion per observable outcome. Label them UAT-AC-01-01-A, UAT-AC-01-01-B, etc.

---

## Section 3 — Regression Scenarios

Define 3–7 behaviors that must never break. These are the "if this fails, something fundamental is wrong" scenarios — the behaviors so central to the system's purpose that their failure indicates a systemic problem, not a feature bug.

**Format per scenario:**

```
### REG-NNN: <short name>

**What it tests:** <one sentence describing the behavior>
**Why it is fundamental:** <what it means for the system if this breaks>
**How to verify:** <human-executable or automated check>
**What could break it:** <the class of change that would cause this to fail>
```

**Criteria for regression scenario selection:**

- Core value proposition: the behavior the system was built to deliver
- Data integrity: the behavior that prevents data loss or corruption
- Authentication boundary: the behavior that prevents unauthorized access
- Integration contract: the behavior that proves the system connects correctly to its most critical dependency

**Do not add regressions that duplicate unit tests.** Regression scenarios are qualitatively different: they verify end-to-end behavior from the user's perspective, not internal implementation details. A regression scenario for a time-tracking CLI is "starting and stopping a timer records accurate elapsed time" — not "the `calculate_duration` function returns correct milliseconds."

---

## Section 4 — Observability Specification

Define what gets logged, in what format, what metrics to instrument, and at what thresholds alerts fire.

### Logging

**Minimum required log events:**

- Request/response at every system boundary (HTTP, queue, external API). Log: timestamp, method, endpoint or operation, status code or result, duration.
- All errors with full context. Log: timestamp, level ERROR, service, trace_id, error message, error type, the input that triggered the error (sanitized — no PII or secrets).
- Key business events. Log: timestamp, level INFO, event name, entity ID, relevant state change. Examples: "order placed", "user registered", "payment processed", "job completed".

**Log events that do not belong:**

- Successful read-only operations on every request (produces noise with no signal)
- Debug traces in production (log at DEBUG level; disable in production unless diagnosing an active incident)
- PII: names, email addresses, phone numbers, payment card data must never appear in logs

**Structured logging format (JSON):**

```json
{
  "timestamp": "2025-06-01T14:23:11.042Z",
  "level": "ERROR",
  "service": "payments",
  "trace_id": "abc123def456",
  "message": "Payment authorization failed",
  "context": {
    "order_id": "ord_789",
    "error_type": "CardDeclined",
    "provider_code": "do_not_honor"
  }
}
```

Every log event carries `timestamp`, `level`, `service`, `trace_id`, and `message`. Additional fields go in `context`. No flat key-value dumping at the top level.

### Metrics

Instrument three tiers:

**System metrics** (infrastructure level — usually provided by platform):
- CPU utilization
- Memory utilization
- Disk I/O

**Service metrics** (application level — instrument in code):
- Request rate (requests per second)
- Error rate (errors / total requests, as a percentage)
- Latency: p50, p95, p99

**Business metrics** (domain level — instrument at business events):
- Define these specific to the project. Examples: orders per minute, new user registrations per hour, job queue depth, payment success rate.

### Alerting Thresholds

Define alert thresholds before deployment, not after the first incident. Minimum required alerts:

| Metric | Threshold | Window | Action |
|---|---|---|---|
| Error rate | > 1% | 5 minutes | Page on-call |
| p99 latency | > 2x baseline | 5 minutes | Page on-call |
| Job queue depth | > 1,000 | 15 minutes | Notify team |
| Deployment error rate | > 0.5% | 2 minutes post-deploy | Trigger rollback review |

Establish a baseline for latency during load testing or the first week of production. "2x baseline" is not measurable without a baseline.

---

## Traceability Summary

Include a traceability table at the end of QA.md mapping every SPEC.md acceptance criterion to its UAT entry:

| AC ID | AC Summary | UAT Criterion | Regression Scenario |
|---|---|---|---|
| AC-01-01 | User can log in with email/password | UAT-AC-01-01 | REG-001 (auth boundary) |
| AC-02-03 | Timer records elapsed time accurately | UAT-AC-02-03 | REG-002 (core value prop) |

Acceptance criteria with no UAT mapping are flagged as gaps requiring clarification.

---

## Common Mistakes

**Test matrix with 100% unit coverage target.** This is unrealistic and counterproductive. 100% line coverage requires testing trivial getters, error paths that cannot be triggered, and framework boilerplate. It incentivizes meaningless tests written to satisfy the number. 80% on business logic is the right target.

**UAT criteria requiring code knowledge to execute.** "Verify that `UserSession.is_valid()` returns `True` after login" is not UAT — it is a unit test phrased awkwardly. UAT is executed by a human with no access to the code. Translate every criterion to: given a state the tester can establish via the UI or CLI, what do they do, and what do they observe?

**Observability that logs everything or nothing.** Logging every database call at INFO level produces noise that buries the signal. Logging nothing produces blindness. Log at system boundaries, on errors, and on key business events. That is sufficient for operations and incident response.

**No regression scenarios.** A QA.md with a test matrix and UAT criteria but no regression scenarios leaves "what must never break" undefined. The first time something fundamental breaks, the team discovers it is not in anyone's test suite.

---

## Worked Example: qa-strategy for the Time-Tracking CLI

**Test Matrix (partial):**

| Component | Type | Tool | Coverage Target | Owner |
|---|---|---|---|---|
| Timer logic | Unit | pytest | 85% line coverage | dev-agent |
| Storage (SQLite) | Integration | pytest + sqlite3 | All CRUD operations | dev-agent |
| CLI interface | E2E | subprocess + assertions | All documented commands | qa-agent |

**UAT Criteria (2 examples):**

```
### UAT-AC-02-01: Start a timer

**Source:** AC-02-01 — User can start a timer for a named task
**Given:** The CLI is installed. No timer is currently running.
**Action:** Run `timecli start "writing tests"`
**Expected outcome:** The terminal prints: "Timer started: writing tests (HH:MM:SS)"
**Pass signal:** The output contains the task name and a time in HH:MM:SS format.

### UAT-AC-02-03: Timer records accurate elapsed time

**Source:** AC-02-03 — Elapsed time is recorded accurately to the second
**Given:** A timer was started 60 seconds ago (verified by system clock).
**Action:** Run `timecli stop`
**Expected outcome:** The terminal prints the task name and an elapsed time between
  0:01:00 and 0:01:02.
**Pass signal:** Elapsed time is within 2 seconds of actual elapsed time.
```

---

## Writing the Artifact

When all four sections are complete, write the artifact to disk:

- **Path:** `.planning/QA.md` at the project root
- Confirm with: `QA.md written to .planning/QA.md`

## References

- [`references/test-matrix-patterns.md`](references/test-matrix-patterns.md) — How to populate the test matrix: unit, integration, and E2E test candidates; coverage threshold guidance; how to choose the right tool for each component type.
- [`references/uat-criteria-guide.md`](references/uat-criteria-guide.md) — How to write human-executable UAT scenarios from Given/When/Then acceptance criteria, the distinction between AC and UAT, 5 complete AC-to-UAT translation examples.
- [`references/observability-spec.md`](references/observability-spec.md) — Structured logging format, the 3-tier metric hierarchy, alerting threshold patterns, and the operational distinction between logging for debugging vs. logging for operations.
