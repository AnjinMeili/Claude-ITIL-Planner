# Debt Management Reference

Debt classification framework, DEBT.md entry format, triage matrix, and the line between acceptable debt and unacceptable defects.

---

## Debt Classification

### Intentional Debt

A deliberate, time-bounded trade-off made with full awareness of the consequences. The team understands what is being deferred, why it is being deferred, and under what conditions it must be resolved.

**Hallmarks of intentional debt:**
- A trigger condition or resolution date exists at the time the debt is incurred
- The impact of non-resolution is understood and accepted by decision-makers
- The workaround is documented at the point of use (a comment referencing the DEBT.md entry)

**Example:**
> We are using SQLite for the session store at launch. It will not survive horizontal scaling. We accept this because we do not expect to need more than one instance until 10k DAU. Resolution trigger: >5k DAU sustained for 7 days.

### Unintentional Debt

A gap or deficiency discovered after the fact — not a deliberate trade-off, but an oversight revealed by usage, review, or failure.

**Hallmarks of unintentional debt:**
- Not known at time of implementation
- Discovered through a bug report, a code review, or hitting a production edge case
- No deliberate decision was made to defer it

**Example:**
> The session expiry check assumes the server clock is UTC. Discovered when a user in UTC-8 reported sessions expiring unexpectedly. Not a deliberate trade-off — a gap in our time handling assumptions.

### Bit Rot

Previously correct code that has become incorrect or degraded due to changes in the surrounding environment: dependency upgrades, infrastructure changes, evolving external APIs, or shifting requirements.

**Hallmarks of bit rot:**
- Code was correct when written
- An external change — not a code change — caused the degradation
- The code still runs but produces incorrect or suboptimal results

**Example:**
> The billing module uses the Stripe v2 PHP API client. Stripe deprecated v2 in 2023 and will remove it in 2025. The code runs today but will break at the EOL date.

---

## DEBT.md Entry Format

```markdown
## DEBT-NNN: <short title>

**What:** <one sentence describing the gap, workaround, or degraded state>
**Why deferred:** <one sentence explaining the reason — timeline, complexity, dependency, etc.>
**Impact:** <what breaks or degrades if this is never resolved; be specific>
**Category:** intentional | unintentional | bit-rot
**Target resolution:** <milestone label, date, or "when <condition> is met">
**Opened:** YYYY-MM-DD
**Resolved:** — (replace with date and brief note when closed)
```

**Numbering:** Assign sequential IDs (`DEBT-001`, `DEBT-002`). Never reuse a closed ID.

**Reference from code:** At the point where the workaround lives, add a comment:

```python
# DEBT-042: Using in-memory cache; will not survive process restart.
# Replace with Redis when deploying multi-instance. See DEBT.md.
result = _in_memory_cache.get(key)
```

This makes the debt discoverable from the code, not only from the DEBT.md file.

---

## Example Entries

### Intentional Debt

```markdown
## DEBT-007: In-process job queue

**What:** Background jobs run in a thread pool within the application process rather than
an external queue (e.g., Celery, RQ).
**Why deferred:** External queue infrastructure adds operational complexity we do not need
at current job volume (<100/hour).
**Impact:** Job failures do not survive application restarts. Under high load, the job
thread pool contends with request handling threads.
**Category:** intentional
**Target resolution:** When job volume exceeds 1,000/hour sustained over 24h, or when
the first production incident attributable to in-process queue behavior occurs.
**Opened:** 2025-01-15
**Resolved:** —
```

### Unintentional Debt

```markdown
## DEBT-012: Session clock assumes UTC

**What:** Session expiry calculation uses `datetime.now()` (local time) rather than
`datetime.utcnow()` or timezone-aware datetimes. Sessions expire at incorrect times
for users in non-UTC timezones.
**Why deferred:** Discovered post-launch; fix requires migrating existing session records.
**Impact:** Sessions for users in UTC- timezones expire earlier than intended. Sessions
for users in UTC+ timezones persist longer than intended.
**Category:** unintentional
**Target resolution:** 2025-Q2 milestone (before international user expansion).
**Opened:** 2025-02-03
**Resolved:** —
```

### Bit Rot

```markdown
## DEBT-019: Stripe v2 API client approaching EOL

**What:** The billing module uses stripe-php v2 client library. Stripe has announced
EOL for v2 support on 2025-09-30.
**Why deferred:** v3 migration requires changes to webhook signature verification and
PaymentIntent handling. Scoped for a dedicated migration sprint.
**Impact:** After 2025-09-30, Stripe may reject API calls from v2 clients. Billing
will break.
**Category:** bit-rot
**Target resolution:** 2025-08-01 (one month before EOL to allow buffer).
**Opened:** 2025-03-01
**Resolved:** —
```

---

## Impact × Effort Triage Matrix

Use this matrix at each debt triage session to prioritize resolution order.

```
             LOW EFFORT    HIGH EFFORT
HIGH IMPACT  [Resolve now] [Plan sprint]
LOW IMPACT   [Quick win]   [Defer or close]
```

**High Impact / Low Effort:** Resolve in the current sprint. These are the debts that are costing more to carry than to fix.

**High Impact / High Effort:** Schedule a dedicated sprint or milestone. Document the migration plan. Do not let these age indefinitely — set a hard deadline.

**Low Impact / Low Effort:** Resolve when the surrounding code is touched anyway. Do not make a special trip.

**Low Impact / High Effort:** Defer. Revisit at the next major version or close the item if the code path is slated for replacement.

Triage debt at the start of each milestone. For each open item: assess whether the impact has changed (bit rot tends to increase in impact as EOL dates approach), assess whether the effort has changed (refactoring in adjacent areas may make a previously expensive fix cheap), and update `Target resolution` accordingly.

---

## Debt That Is Never Acceptable to Carry

The following do not belong in DEBT.md — they are defects that block shipping:

**Known security vulnerabilities**
- A dependency with a critical CVE in an exploited attack surface
- An authentication bypass discovered in review
- An exposed credential anywhere in the codebase
- An injection vulnerability (SQL, command, path traversal) in any code path reachable from user input

**Known data corruption risks**
- A race condition that can produce incorrect records under concurrent writes
- A missing transaction on a multi-step write where partial completion produces corrupt state
- An incorrect delete scope that can remove records it should not
- A migration that can silently lose data

Recording these in DEBT.md implies they are acceptable to deploy. They are not. Fix them before the code reaches any environment with real data. If the fix requires significant effort, scope it as the next immediate work item and do not ship without it.

---

## Triage Checklist

At the start of each milestone, run through DEBT.md:

- [ ] Has any item's trigger condition been met? → Schedule resolution this milestone.
- [ ] Has any item's impact increased? → Re-evaluate priority.
- [ ] Has any item's EOL date come within 90 days? → Escalate to High Impact.
- [ ] Are any items in the "never acceptable" categories that somehow got into DEBT.md? → Fix immediately; remove from DEBT.md.
- [ ] Are any items resolved but not closed? → Update `Resolved` field and add a brief note.
- [ ] Are any items no longer relevant (code path removed, feature abandoned)? → Close with reason.
