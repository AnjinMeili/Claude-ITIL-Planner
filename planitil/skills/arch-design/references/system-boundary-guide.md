# System Boundary Guide

Defining system boundaries is the first step of ARCH.md. Boundaries determine what
is inside the system (owned, changeable), what is outside (depended upon, not owned),
and where the interfaces between them are.

---

## System Context: What Belongs

A system context diagram answers one question: **what external actors and systems
interact with this system?**

External actors fall into four categories:

| Category | Examples | Relationship |
|---|---|---|
| Human users | End users, admins, operators | Initiate requests, consume responses |
| Upstream systems | Authentication providers, payment processors | Provide data or services this system depends on |
| Downstream systems | Analytics pipelines, notification services, audit logs | Consume events or data this system produces |
| Peer systems | Other services in the same product | Bidirectional data exchange |

**What to exclude from the context diagram:**
- Internal components (those belong in the component diagram)
- Implementation details (language, framework, library choices)
- Future integrations not yet in scope

**Context diagram format (text):**

```
[External Actor] --[interaction description]--> [This System]
[This System] --[interaction description]--> [External System]
```

Example for a time-tracking CLI:
```
[Developer] --starts/stops tasks, views reports--> [timetrack CLI]
[timetrack CLI] --reads/writes entries--> [~/.timetrack/entries.db]
[timetrack CLI] --exports CSV--> [Developer's filesystem]
```

---

## Internal vs External: The Ownership Test

A component is **internal** if the team can change it without coordinating with another team.
A system is **external** if changing its behavior requires a conversation with someone else.

When in doubt: if it has its own SLA, it is external.

Common misclassifications:
- A shared internal library owned by your team: **internal** (you can change it)
- A shared internal library owned by another team: **external** (you must negotiate changes)
- A database instance: **internal** (you control the schema)
- A managed database service (RDS, Firestore): the service itself is **external**, the schema is **internal**

---

## Interface Contract Format

Every boundary crossing needs an interface contract. A contract defines what crosses
the boundary — not how it is implemented.

**Contract fields:**

```
Interface: [Name — what this interface does]
Direction: [inbound | outbound | bidirectional]
Protocol: [HTTP/REST | gRPC | message queue | file | SDK call | etc.]
Input: [What this interface receives — data shape, key fields]
Output: [What this interface returns — data shape, key fields]
Errors: [What failure cases are possible and how they are signaled]
SLA: [Latency target, availability expectation, rate limit if applicable]
Owner: [Who is responsible for this interface — team or system name]
```

**Example contract (CLI → SQLite):**
```
Interface: Entry persistence
Direction: bidirectional
Protocol: SQLite SQL via stdlib sqlite3
Input: INSERT/UPDATE/SELECT/DELETE statements with parameter binding
Output: Row data as Python dict, rowcount for mutations
Errors: sqlite3.OperationalError (schema mismatch), sqlite3.DatabaseError (corruption)
SLA: <10ms for single-row reads/writes; no external availability dependency
Owner: timetrack CLI (schema owned here)
```

**Example contract (API → Auth Provider):**
```
Interface: Token validation
Direction: inbound (we call them)
Protocol: HTTPS POST to /oauth/introspect
Input: {"token": "<access_token>"}
Output: {"active": true/false, "sub": "<user_id>", "scope": "<scopes>"}
Errors: HTTP 401 (invalid client credentials), HTTP 503 (provider unavailable)
SLA: p99 <200ms per provider SLA; 99.9% monthly uptime
Owner: Auth Provider (external — do not modify)
```

---

## Logical vs Physical Boundaries

A logical boundary is about responsibility: who owns this concern?
A physical boundary is about deployment: where does this code run?

These are often different:

| Logical Boundary | Physical Boundary |
|---|---|
| "Auth service" | Could be a separate process or a module in a monolith |
| "Notification system" | Could be an async queue or a synchronous function call |
| "Reporting" | Could be the same database or a read replica |

**Design logical boundaries first.** Physical deployment is a later decision (often an ADR).
Mixing logical and physical decisions in the same diagram creates confusion.

---

## Common Boundary Mistakes

### Too Granular
**Symptom:** Every function or class becomes a "component" in the diagram.
**Problem:** The diagram no longer communicates architecture — it is a class diagram.
**Fix:** Group by responsibility (what problem it solves), not by file structure.

### Too Coarse
**Symptom:** One box labeled "Backend" for everything server-side.
**Problem:** No meaningful interface contracts can be drawn; the diagram communicates nothing.
**Fix:** Split along the lines of ownership, deployment unit, or distinct SLA.

### Missing External Dependencies
**Symptom:** Diagram shows only internal components, no external systems.
**Problem:** The hardest failure modes (external system unavailability, API changes) are invisible.
**Fix:** Every external API, storage service, or auth provider must appear on the diagram.

### Interface Without Contract
**Symptom:** An arrow between two boxes with no label or a vague label ("calls", "uses").
**Problem:** The interface is implicit — behavior changes will not be caught until runtime.
**Fix:** Every arrow has a contract. If you cannot write the contract, the boundary is not real.

### Circular Dependencies
**Symptom:** Component A calls Component B and Component B calls Component A.
**Problem:** Neither can be tested in isolation; changes to either affect both.
**Fix:** Introduce a mediator, invert one dependency, or merge the components.
