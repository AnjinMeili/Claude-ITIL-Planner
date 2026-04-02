---
name: arch-design
description: |
  Produces ARCH.md and ADR files for a project. Defines system boundaries, component relationships, data model, interface contracts, and deployment model. Records each significant architectural decision with rationale and rejected alternatives.

  <example>
  User says "let's design the system architecture" or "what's the right structure for this" — invoke arch-design to produce ARCH.md and the first set of ADRs before implementation planning begins.
  </example>

  <example>
  After SPEC.md is confirmed and use cases are stable, before implementation planning begins — invoke arch-design to translate requirements into a structural blueprint that implementation planning can reference.
  </example>
version: 1.0.0
---

# arch-design

Produce a structural blueprint for the system: ARCH.md as the living overview, and individual ADR-NNN.md files for each significant architectural decision. Design to the requirements in SPEC.md — no more, no less.

---

## What This Skill Produces

### ARCH.md

ARCH.md is the system overview. It answers the question "what is this system and how does it work" at an architectural level. It is not a feature list, a roadmap, or a project plan. It describes structure.

Sections:

**System Context** — Identify every external system, user type, or data source that interacts with the system. Show the boundary: what is inside this system, what is outside, and what crosses the boundary. A system context diagram (described in text or as an ASCII diagram) should be legible to any engineer without prior knowledge of the project.

**Component Diagram** — List the major internal components, what each one is responsible for, and how they communicate. Components are logical units, not files or classes. Show the relationships: which components call which, which share data, which are synchronous vs asynchronous. Do not list every module — show the units that matter for understanding the system's shape.

**Data Model** — Identify the key entities, their attributes at the level that matters for architecture (not every field), and the relationships between them. If there are important constraints (e.g., "an Invoice must have at least one LineItem") include them. The goal is to communicate structure, not to serve as a database schema.

**Interface Contracts** — Define the contract at each major component boundary: input format, output format, error conditions, and any SLA or throughput expectation. Interface contracts describe what a caller can expect, not how the callee implements it. An interface contract for a function specifies its signature and behavior; for a service, its protocol, endpoint shape, and error codes.

**Deployment Model** — Describe where components run: local process, server, container, serverless function, embedded in another process. Identify any infrastructure dependencies (databases, queues, caches). Show how components reach each other at runtime.

---

### ADR-NNN.md Files

Produce one ADR for each significant architectural decision. File naming: ADR-001.md, ADR-002.md, and so on. Store in the same directory as ARCH.md (typically `.planning/`).

The ADR format:

```
# ADR-NNN: [Title]

**Status:** Proposed | Accepted | Superseded by ADR-NNN

## Context
[The forces that led to this decision. What is the situation, what constraints apply, what goals are in tension.]

## Decision
[What was chosen. One clear statement of the decision.]

## Consequences
### Easier
[What becomes easier or better as a result of this decision.]

### Harder
[What becomes harder or worse. Be honest — every architectural decision has a downside.]

## Alternatives Rejected

### [Alternative A]
[Why this alternative was not chosen.]

### [Alternative B]
[Why this alternative was not chosen.]
```

The Alternatives Rejected section is mandatory. A decision without rejected alternatives is not a decision — it is an assumption. If only one option was ever considered, that is a signal to widen the search before marking the ADR Accepted.

---

## When to Create an ADR

Create an ADR when the decision meets any of these criteria:

- **Hard to reverse** — changing it later would require significant rework (e.g., storage format, wire protocol, authentication model)
- **Affects multiple components** — two or more components are shaped by the choice
- **Involves a trade-off between valid options** — there is no objectively correct answer; reasonable engineers could disagree
- **Has significant cost, performance, or operational implications** — the choice constrains how the system scales, how it fails, or what it costs to run

Do not create ADRs for implementation details — those belong in code comments or design notes. ADRs record architectural choices, not code patterns.

---

## Integration: failure-modes

Before finalizing ARCH.md, invoke the failure-modes skill with architecture type context. Failure-modes analysis will surface the ways the proposed architecture can go wrong under load, partial failure, or misuse. Address each identified failure mode in ARCH.md — either by adjusting the design or by documenting the known risk and the accepted trade-off.

Do not skip this step for systems that have external dependencies, shared mutable state, or asynchronous communication between components.

---

## Integration: confidence-gate

Run confidence-gate before marking any ADR as Accepted. Confidence-gate evaluates whether the decision has been adequately explored — alternatives considered, consequences documented, and reviewers satisfied. An ADR that has not passed confidence-gate stays in Proposed status.

---

## Common Mistakes

**ARCH.md as a feature list.** A feature list describes what the system does. ARCH.md describes how it is structured. If the Component Diagram section reads like a product backlog, start over. Ask: what are the major structural units and how do they relate?

**ADRs without rejected alternatives.** If every ADR shows only the chosen option, the architecture has not been designed — it has been assumed. Go back and enumerate at least the most obvious alternative for each decision, and explain why it was not chosen.

**Interface contracts that describe implementation.** An interface contract is a promise to callers. It should not mention how the callee is implemented. If the interface contract for a storage component mentions "uses SQLite internally," remove it. If removing it makes the contract unclear, the contract is probably incomplete.

**Data model that includes every field.** The architectural data model is not a database schema. Include the entities, their key attributes, and their relationships. Leave column-level detail to the implementation. If the data model has more than 20 fields per entity, it is too detailed.

**System context that omits external dependencies.** Every external system, API, human actor, or data source that crosses the system boundary belongs in the system context. Omitting them does not make them go away — it makes the system's dependencies invisible.

---

## Worked Example: Time-Tracking CLI

Applying arch-design to a time-tracking CLI that lets developers log work sessions from the terminal and generate weekly reports.

### ARCH.md Outline

**System Context**

External actors: Developer (terminal user), local filesystem (storage), optional remote sync endpoint (future, out of scope for v1). The system boundary encloses the CLI binary and the local data store.

**Component Diagram**

```
[CLI Parser] --> [Command Handler]
[Command Handler] --> [Session Store]
[Command Handler] --> [Report Engine]
[Session Store] --> [Storage Adapter]
[Storage Adapter] --> [Local Filesystem]
```

CLI Parser: parse argv, validate command structure, route to handler.
Command Handler: orchestrate business logic for start, stop, log, report commands.
Session Store: manage session lifecycle (open, close, query).
Report Engine: aggregate sessions into weekly summaries.
Storage Adapter: read/write the persistent data format.

**Data Model**

- Session: id, project, start_time, end_time (nullable), tags
- Project: id, name, default_tags
- A Session belongs to one Project. A Project has zero or more Sessions.

**Interface Contracts**

Storage Adapter interface:
- `write(session: Session) -> Result<(), StorageError>`
- `read_all() -> Result<Vec<Session>, StorageError>`
- `read_range(start: DateTime, end: DateTime) -> Result<Vec<Session>, StorageError>`
- Error conditions: file not found (returns StorageError::NotFound), parse error (StorageError::Corrupt), permission denied (StorageError::PermissionDenied)

**Deployment Model**

Single binary, runs in a local terminal. Data stored in `~/.timetracker/` by default, configurable via `TIMETRACKER_DATA_DIR`. No network access in v1.

---

### ADR-001: Storage Format

**Status:** Accepted

**Context**

The CLI needs to persist session data locally. The data volume is small (hundreds to low thousands of records). The system must support range queries by date. The format must be readable without special tooling for debugging and data portability. No multi-user or concurrent access is required.

**Decision**

Use a single JSONL file (one JSON object per line) as the storage format.

**Consequences**

Easier: trivially readable with any text editor or `jq`; no schema migration needed for additive changes; portable across platforms; no dependency on a database library.

Harder: no indexed queries — range queries require full file scan; concurrent writes (if ever needed) require file locking; very large datasets will be slow to query (acceptable given the use case).

**Alternatives Rejected**

SQLite: provides indexed queries and concurrent-safe writes. Rejected because it adds a native dependency (complicates cross-platform builds), requires schema migrations, and is not human-readable without tooling. The query performance benefit is not needed at the expected data volume.

Remote API: offloads storage to a server. Rejected because it requires network access, an account, and a server, all of which are out of scope for a local CLI tool whose primary value is simplicity and offline use.

---

## References

- `references/adr-patterns.md` — Guide to writing effective ADRs: Nygard format, Context section, enumerating alternatives, writing Consequences, and three full example ADRs.
- `references/system-boundary-guide.md` — Guide for defining system boundaries: context diagrams, external vs internal, interface contract format, logical vs physical boundaries, common mistakes.
- `references/assets/ADR.md.template` — Fill-in-the-blank ADR template with all sections and guidance comments.
