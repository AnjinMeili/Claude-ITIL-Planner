---
name: requirements-spec
description: |
  Use when a confirmed PROJECT.md exists and the next step is defining what the system must do before architecture or planning begins. Produces SPEC.md, the structured contract between problem understanding and solution design.

  <example>
  User: "Let's define the requirements for the time-tracking CLI"
  Action: Run requirements-spec to produce SPEC.md with traceable use cases, Given/When/Then acceptance criteria, non-functional requirements, and external dependencies — before any architecture or implementation planning starts.
  </example>

  <example>
  User: "What should the system do?" or "Let's write up the feature requirements"
  Action: Run requirements-spec to translate PROJECT.md's problem statement and success metrics into structured use cases with testable acceptance criteria and traceable IDs that PLAN.md tasks will reference.
  </example>
version: 1.0.0
---

# requirements-spec

Produce SPEC.md — the structured bridge between PROJECT.md (what problem) and ARCH.md (how to solve it). Every use case, acceptance criterion, and non-functional requirement in this document must be traceable to PROJECT.md and testable before release.

## What This Skill Produces

A single SPEC.md file with four sections:

1. **Use Cases** — actor-action-outcome statements with traceable IDs
2. **Feature Acceptance Criteria** — Given/When/Then criteria per use case
3. **Non-Functional Requirements** — performance, security, reliability, observability
4. **External Dependencies** — every external system with API, SLA, failure mode, and owner

## Prerequisites

SPEC.md requires a confirmed PROJECT.md. Before writing a single use case, verify:

- Problem statement is confirmed (no solution language)
- Primary audience is named (the UC actor comes from here)
- Success metrics are accepted (acceptance criteria must map to these)
- Constraints are listed (NFRs must respect these)

If PROJECT.md is not confirmed, stop and run project-charter first.

## Section 1: Use Cases

### Format

Each use case follows the actor-action-outcome pattern:

> As [specific actor from TARGET AUDIENCE in PROJECT.md], I need to [action — what they do, not how the system does it] so that [outcome — the value they receive].

The actor is never "the system" or "the application." The actor is always a human or an external system with agency. Pull actors directly from the Target Audience section of PROJECT.md.

### IDs

Assign each use case a sequential ID: UC-01, UC-02, UC-03. These IDs appear in PLAN.md tasks so that implementation work traces back to requirements.

### Scope

Write 3–10 use cases. Fewer than 3 means the scope is not understood. More than 10 means the project is not bounded or use cases are too granular.

Use cases cover primary workflows — the paths a user takes to accomplish the goals described in PROJECT.md's Target Audience section. Edge cases and error paths appear in acceptance criteria, not as separate use cases.

### What Belongs in a Use Case vs. Acceptance Criterion

A use case names the goal. Acceptance criteria define the boundary conditions under which the goal is met or failed. Do not write use cases that are acceptance criteria in disguise.

Use case (goal-level): "As a billing engineer, I need to start a timer when beginning a task so that time is recorded without manual intervention."
Acceptance criterion (condition-level): "Given I am in a terminal, when I run `track start PROJ-123`, then a timer begins and the timestamp is stored locally."

## Section 2: Feature Acceptance Criteria

### Format

Each acceptance criterion uses Given/When/Then:

- **Given** — precondition (system state, user state, data that exists)
- **When** — triggering action (what the user or system does)
- **Then** — observable outcome (what is verifiably true afterward)

Every Then clause must be verifiable by automated test or by a defined manual test procedure. If confirming a Then clause requires human judgment ("the interface looks good," "the response is reasonable"), rewrite it until it is objective.

### IDs

Acceptance criteria inherit the use case ID with a numeric suffix: AC-01-01 is the first criterion for UC-01, AC-01-02 is the second, and so on. These IDs appear in QA.md test cases.

### Quantity

Write 2–5 acceptance criteria per use case. Two is the minimum (happy path plus at least one failure path). More than 5 usually signals a use case that should be split.

### Mapping to Test Types

Each criterion implicitly belongs to a test type:
- **Unit** — Then clause is verifiable in isolation with no external system
- **Integration** — Then clause requires two or more components or an external API
- **End-to-end** — Then clause requires a full user journey from input to observable outcome

Label each criterion with its test type. QA.md uses these labels to assign test implementation ownership.

## Section 3: Non-Functional Requirements

NFRs constrain how the system behaves across all use cases. They are not features — they are properties of the system that must hold under all conditions.

### Performance

State explicit thresholds. Vague NFRs ("the system should be fast") are not requirements.

Specify:
- Latency targets (p50, p95, p99) under defined load conditions
- Throughput requirements (requests per second, records per minute)
- Startup time for CLI tools or services
- Any hard limits from PROJECT.md constraints

### Security

State authentication and authorization requirements explicitly:
- How users authenticate (and which use cases require authentication)
- What authorization model applies (who can read, write, delete what)
- Data handling requirements (what is stored, where, for how long)
- Secrets management (how credentials are stored and rotated)

Do not assume security requirements are obvious. State them.

### Reliability

Define the availability contract:
- Uptime target (if applicable)
- Acceptable error rate under normal conditions
- Recovery behavior after failure (data loss tolerance, retry policy)
- Degraded-mode behavior when external dependencies are unavailable

### Observability

Specify what the system must expose for operational visibility:
- What events are logged (and at what level)
- What metrics are emitted (and to what sink)
- What user-facing error messages exist (and what information they include)
- How a user or operator diagnoses a failure without reading source code

## Section 4: External Dependencies

List every external system this project integrates with. "External" means anything outside the codebase boundary: third-party APIs, internal services owned by other teams, databases not managed by this project, authentication providers.

For each dependency, document four things:

| Field | What to capture |
|---|---|
| API / Protocol | The specific API version or protocol used |
| SLA | The availability and latency guarantees the dependency provides |
| Failure mode | What happens in this project when the dependency is unavailable |
| Owner | The team or vendor responsible for this dependency |

Failure mode is the most commonly omitted field. Define it before architecture begins — it drives resilience design decisions in ARCH.md.

## Traceability

Every element in SPEC.md must trace forward and backward:

- Each use case traces back to a goal in PROJECT.md's Target Audience section
- Each acceptance criterion traces back to its use case
- Each non-functional requirement traces back to a constraint or success metric in PROJECT.md
- PLAN.md tasks reference use case IDs (UC-01) and criterion IDs (AC-01-02) so implementation work stays connected to requirements

When a task cannot be traced to any ID in SPEC.md, it is either undocumented scope or out of scope. Neither situation is acceptable — resolve before planning continues.

## Worked Example

Continuing from project-charter's "developer time-tracking CLI":

---

**UC-01:** As a freelance backend engineer, I need to start a timer from my terminal when beginning a task so that time is recorded without interrupting my workflow.

**AC-01-01 (Unit):** Given the CLI is installed, when I run `track start PROJ-123`, then a timer entry is created in local storage with the current timestamp and the task ID.

**AC-01-02 (Unit):** Given a timer is already running, when I run `track start PROJ-456`, then the running timer stops, its end timestamp is recorded, and a new timer starts for PROJ-456.

**AC-01-03 (Unit):** Given no authenticated session exists, when I run `track start PROJ-123`, then the CLI outputs an authentication error and exits with a non-zero code without writing any data.

---

**UC-02:** As a freelance backend engineer, I need to submit accumulated time entries to Jira so that client billing reflects actual work without manual copy-paste.

**AC-02-01 (Integration):** Given at least one local time entry exists for a Jira project, when I run `track sync`, then all unsynced entries are submitted to the corresponding Jira tickets and marked as synced in local storage.

**AC-02-02 (Integration):** Given the Jira API is unreachable, when I run `track sync`, then the CLI outputs a connectivity error, no local data is modified, and the command exits with a non-zero code.

**AC-02-03 (End-to-end):** Given a full work session with multiple task switches, when I run `track sync`, then each Jira ticket shows the correct time logged with timestamps accurate to within 60 seconds.

---

## Common Mistakes

**Writing requirements as implementation.** "The system stores time entries in a SQLite database" is a design decision, not a requirement. The requirement is "time entries persist across CLI sessions." Where they are stored is ARCH.md's decision.

**Acceptance criteria that require human judgment.** "The output looks clean" cannot be automated. "The output includes the task ID, elapsed time, and sync status on a single line" can be automated. Rewrite every Then clause until it is objective.

**Omitting NFRs because they seem obvious.** NFRs that are "obvious" are discovered during production incidents. Stating "CLI startup time must be under 500ms" prevents a dependency choice in ARCH.md that introduces a 2-second JVM startup.

**Use cases that are features.** "The system shall support CSV export" is a feature description. The use case is "As a billing manager, I need to export time entries to a spreadsheet so that I can generate client invoices in my existing billing tool."

**Missing external dependency failure modes.** Every external dependency will be unavailable at some point. Designing failure behavior after the architecture is locked is expensive. Define it now.

## Artifact Output

SPEC.md is written to `.planning/SPEC.md` at the project root. The `requirements-analyst` agent handles the write. Confirm output with the analyst's confirmation message before proceeding to arch-design.

## References

- [use-case-patterns.md](references/use-case-patterns.md) — actor-action-outcome format, anti-patterns, and weak→strong rewrites
- [acceptance-criteria-guide.md](references/acceptance-criteria-guide.md) — Given/When/Then format, testability test, and untestable→testable rewrites
- [SPEC.md.template](references/assets/SPEC.md.template) — fill-in-the-blank template with all four sections and ID placeholders
