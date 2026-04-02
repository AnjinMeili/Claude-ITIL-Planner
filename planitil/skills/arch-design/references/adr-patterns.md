# ADR Patterns

Architecture Decision Records (ADRs) document significant decisions with the context
that made them necessary, what was chosen, and — critically — what was not chosen and why.

---

## The Nygard Format

Every ADR follows this structure:

```
# ADR-NNN: [Short imperative title — what was decided]

**Status:** Proposed | Accepted | Superseded by ADR-NNN | Deprecated

## Context
[The forces at play. What problem made this decision necessary? What constraints,
requirements, or competing concerns shaped the decision space? Do not describe the
solution here — only the situation that required a decision.]

## Decision
[What was decided, stated plainly. One or two sentences. The reader should understand
the choice without needing to infer it from the consequences.]

## Consequences
[What becomes easier as a result of this decision. What becomes harder. What new
constraints does this decision create for future decisions. Be honest about the costs —
an ADR that lists only benefits is not credible.]

## Alternatives Rejected
[Each alternative considered, with a clear statement of why it was not chosen.
At minimum, include the most obvious alternative. An ADR without rejected alternatives
documents an assumption, not a decision.]
```

---

## Writing the Context Section

The Context section is the most important and most commonly done poorly.

**What to include:**
- The specific requirement or constraint that made this decision necessary
- Competing forces (e.g., "needs to work offline" vs "needs to sync across devices")
- Any prior decisions that constrain this one
- The consequence of not deciding (if relevant)

**What to exclude:**
- Background history not relevant to the decision
- The solution (that belongs in Decision)
- Opinions about which option is better

**Weak Context:**
> We need to store time entries somewhere. There are several options for storage.

**Strong Context:**
> The CLI must work fully offline (PROJECT.md constraint). Time entries are structured
> records (task, start, stop, tags) requiring occasional queries by date range and tag.
> The team has no server infrastructure. Storage must be embeddable in the CLI binary
> or available as a standard OS component.

---

## Writing the Alternatives Rejected Section

Every alternative entry answers: **what was the option, and what made it insufficient?**

Format each entry as:
```
### [Alternative Name]
[One sentence describing the option]
**Rejected because:** [The specific reason this option did not meet the requirements
or created unacceptable trade-offs]
```

Rules:
- Include at least the most obvious alternative (the one someone reading this would ask about)
- "Too complex" is not a rejection reason — state what complexity cost would not be justified
- "Not mature enough" is not a rejection reason — state what specific capability was missing
- Personal preference is not a rejection reason — ground rejections in requirements

---

## Three Full Example ADRs

### Example 1: Storage Format

```markdown
# ADR-001: Use SQLite for local time entry storage

**Status:** Accepted

## Context
The CLI must work fully offline (PROJECT.md). Time entries are structured records
(task, start_time, stop_time, tags) requiring queries by date range and tag. The
application has no server component. Storage must be queryable without loading the
full dataset into memory, which rules out flat file formats at any reasonable scale
(>10k entries over a year of use).

## Decision
Use SQLite via the standard library `sqlite3` module. The database file is stored
at `~/.timetrack/entries.db`.

## Consequences
Easier: complex queries (date ranges, tag filters, aggregations) without loading all
data; atomic writes prevent corruption on crash; standard tooling for inspection.
Harder: the database file is not human-readable; migration is required when schema
changes; concurrent write access from multiple processes requires WAL mode.

## Alternatives Rejected

### Flat JSON file
Append-only JSON file, one entry per line.
**Rejected because:** Querying by date range or tag requires loading the entire file.
At 10 entries/day, this reaches 3,650 entries/year — acceptable today but
requires a rewrite at scale. SQLite queries remain fast at any realistic size.

### Remote API (hosted database)
Store entries in a cloud service.
**Rejected because:** Breaks the offline requirement (PROJECT.md constraint). Adds
network dependency, authentication complexity, and a monthly cost.
```

### Example 2: API Style

```markdown
# ADR-002: Use REST over HTTP/JSON for the API layer

**Status:** Accepted

## Context
The service exposes data to three consumers: a web dashboard, a mobile app, and
third-party integrations (SPEC.md UC-07). All consumers run on different platforms.
The team has strong HTTP/REST experience. The API will be public and versioned.

## Decision
Expose a REST API over HTTP with JSON request/response bodies. API versioning via
URL prefix (`/v1/`).

## Consequences
Easier: all planned consumers already have HTTP client libraries; REST is universally
understood; curl-testable without special tooling; straightforward to document.
Harder: no built-in schema contract enforcement; versioning requires discipline;
N+1 query patterns are possible if resource design is careless.

## Alternatives Rejected

### GraphQL
Single endpoint, client-specified queries.
**Rejected because:** The three consumers have stable, known query patterns —
the flexibility GraphQL provides is not needed and adds resolver complexity,
N+1 query risk, and a steeper learning curve for third-party integrators unfamiliar
with GraphQL clients.

### gRPC
Binary protocol with Protobuf schemas.
**Rejected because:** Third-party integrators (UC-07) expect HTTP/JSON. gRPC
requires generated client stubs, which raises the integration barrier for external
developers. The performance advantage of binary encoding is not justified by the
data volumes in this system.
```

### Example 3: Authentication Strategy

```markdown
# ADR-003: Use JWT with refresh token rotation for session management

**Status:** Accepted

## Context
The API requires authenticated access (SPEC.md NFR-02). Sessions must survive
browser restarts (persistent login). The system has no server-side session store.
Tokens must be revocable in the event of compromise.

## Decision
Issue short-lived JWTs (15-minute expiry) with long-lived refresh tokens (30-day
expiry, rotation on use). Store refresh tokens in the database for revocation.

## Consequences
Easier: stateless JWT validation on every request without database lookup;
revocation works via refresh token invalidation; standard libraries available.
Harder: 15-minute access token window means stale data is served up to 15 minutes
after a permissions change; refresh token rotation requires the client to handle
concurrent-request race conditions; token storage on mobile requires secure enclave.

## Alternatives Rejected

### Sessions with server-side store (Redis)
Server stores session state; client holds only a session ID.
**Rejected because:** Requires a Redis dependency not in the current stack. Adds
operational complexity (Redis availability affects all authenticated requests).
Rejected given that the JWT approach achieves revocability without a session store.

### Long-lived JWT (no refresh)
Single JWT with 30-day expiry, no refresh mechanism.
**Rejected because:** A compromised token cannot be revoked until expiry. 30 days
is an unacceptable exposure window for a token that grants full account access.
```

---

## ADR Creation Threshold

Create an ADR when the decision meets at least one of:

| Criterion | Example |
|---|---|
| Hard to reverse | Database choice, API protocol, authentication scheme |
| Affects 2+ components | Shared data format, inter-service communication pattern |
| Meaningful trade-off | Two valid options with different strengths |
| Significant cost/performance implication | Caching strategy, indexing approach |

Do NOT create ADRs for:
- Implementation details (variable naming, internal function structure)
- Decisions with only one viable option (no real trade-off)
- Temporary decisions clearly marked for later revisitation
- Choices fully constrained by prior ADRs
