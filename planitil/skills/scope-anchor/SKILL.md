---
name: scope-anchor
description: |
  Use when starting any planning task, feature addition, architectural decision, or refactor. Forces explicit scope definition before any planning, architecture, or code work begins. Produces a SCOPE.md artifact that gates all downstream work. Addresses scope ambiguity — the primary root cause of failed AI-generated outputs. Examples:

  <example>
  Context: User is proposing a new greenfield tool. No plan or scope exists yet.
  user: "Let's build a CLI tool that monitors disk usage"
  assistant: "I'll invoke scope-anchor before proceeding. Before we plan or write any code, I need to anchor scope — who this is for, what platform it targets, what's in vs out of scope, and what success looks like. Let me produce a SCOPE.md now."
  <commentary>
  The request uses "build" and describes a new deliverable — a textbook scope-anchor trigger. Without scope definition, the implementation will make unverified assumptions about output format, threshold behavior, OS targets, and alerting mechanism.
  </commentary>
  </example>

  <example>
  Context: User wants to restructure an existing module. The word "restructure" implies meaningful surface area change.
  user: "I need to restructure the auth module"
  assistant: "I'll run scope-anchor first. Restructuring the auth module could mean anything from renaming files to changing the session model — I need to pin down exactly what's in scope, what's out, and what done looks like before touching anything."
  <commentary>
  Refactors that touch more than one file or introduce a new abstraction are explicit scope-anchor triggers. "Restructure" is intentionally vague and carries high risk of silent scope expansion.
  </commentary>
  </example>
version: 1.0.0
---

# scope-anchor

## Purpose

Scope ambiguity kills more AI-assisted work than capability gaps. When the boundary of a task is undefined, planning optimizes for the wrong problem, code covers the wrong surface area, and success becomes unmeasurable. `scope-anchor` is a prerequisite gate: no plan, no architecture document, and no code is produced until SCOPE.md exists and has been confirmed.

This skill applies before work begins, not during it. It takes three to ten minutes. It saves hours.

---

## When to Invoke

Invoke `scope-anchor` at the start of:

- Any planning task or requirements gathering session
- Feature additions to an existing codebase
- Architectural decisions (choosing a pattern, restructuring a module, migrating a system)
- Refactors that touch more than one file or introduce a new abstraction
- Any task where the word "design", "build", "implement", "create", or "migrate" appears in the request

**Do not invoke** for:

- Trivial single-file edits with a fully contained change (e.g., fix a typo, rename a variable)
- Exploratory research where the output is a document, not a deliverable
- Bug fixes where scope is already fully contained in a reproduction case — the bug defines the scope

If in doubt, invoke it. A SCOPE.md that takes three minutes to write is always cheaper than a plan built on an unstated assumption.

---

## The Four Mandatory Questions

Every SCOPE.md answers exactly these four questions. No question is optional. Vague answers are rewritten until they are specific.

### 1. Who is this for?

Identify the target audience or consumer. This is not always a human user. The consumer might be a developer integrating a library, an agent calling a tool, an internal team reading a document, or an automated pipeline ingesting output.

State it explicitly:

- "TypeScript developers building CLI tools with Node 20+"
- "The orchestration agent that reads PLAN.md artifacts"
- "The on-call engineer triaging a production alert"

"Everyone" and "users" are not valid answers. If the audience is broad, name the primary audience and note secondary audiences explicitly.

### 2. What platform, language, framework, or system?

Name the runtime, language, and framework. Include version constraints where they affect behavior. Include the deployment target if relevant.

Examples:

- "Python 3.11, FastAPI, deployed to AWS Lambda"
- "TypeScript, Claude Agent SDK, Node 22, CLI invocation only"
- "Bash scripts running in GitHub Actions on ubuntu-latest"

If the answer is "it depends on what the user has", say that explicitly and describe the detection or negotiation strategy.

### 3. What is explicitly IN scope vs OUT of scope?

List both sides. The out-of-scope list is as important as the in-scope list. Name things that are adjacent, tempting, or commonly assumed — and explicitly exclude them if they are out of scope.

Format as two parallel lists:

**In scope:**
- Specific deliverable A
- Specific deliverable B

**Out of scope:**
- Feature X (deferred to v2)
- Integration Y (owned by another team)
- Edge case Z (acceptable known limitation)

Leaving the out-of-scope list empty is a signal that the scope has not been fully defined. Something is always out of scope.

### 4. What does success look like?

Define success in terms that can be verified. Avoid adjectives. Use observable outcomes.

Weak: "The feature works correctly and is easy to use."
Strong: "A developer can invoke the CLI with no prior documentation and reach a working state in under five minutes. All existing tests pass. The output format matches the schema in `docs/output-schema.json`."

At minimum, success criteria must answer: how will we know this is done, and who confirms it?

---

## SCOPE.md Format

Write SCOPE.md at the root of the relevant working context (project root, planning directory, or feature branch root). Use this structure:

```markdown
# Scope: [Task or Feature Name]

## Audience
[Answer to Question 1]

## Platform and Stack
[Answer to Question 2]

## Scope Boundaries

### In Scope
- [item]
- [item]

### Out of Scope
- [item]
- [item]

## Success Criteria
[Answer to Question 4 — observable, verifiable outcomes]

## Constraints and Assumptions
[Any constraints that affect design choices: time, team size, existing architecture, non-negotiable dependencies. Any assumptions being made explicitly — so they can be challenged.]
```

The Constraints and Assumptions section is not optional. Every task has constraints. Unstated assumptions are scope debt.

---

## The Gate Rule

SCOPE.md must exist and be confirmed before any of the following artifacts are produced:

- PLAN.md or any task plan
- Architecture diagrams or ADRs
- Implementation code
- Test plans or UAT criteria
- PR descriptions

Confirmation means a human or the requesting agent has reviewed the SCOPE.md and agreed it accurately represents the task. In an automated context where no human is in the loop, confirmation means the scope was derived from and traceable to the original request — and any gaps were surfaced explicitly.

If scope changes mid-execution, update SCOPE.md before continuing. A SCOPE.md that no longer matches the work being done is worse than no SCOPE.md — it creates false confidence.

---

## Common Mistakes

**Treating scope as implied.** The most dangerous scopes are the ones that feel obvious. "We're building a login page" implies nothing about session management, password reset, OAuth, rate limiting, or accessibility. State what is in scope. Let the obvious become explicit.

**Conflating "what to build" with "how to build it."** SCOPE.md answers what and for whom. It does not answer how. Architectural choices, technology selection, and implementation strategy belong in PLAN.md. Mixing them into SCOPE.md produces a document that is too long to confirm and too opinionated to be stable.

**Leaving success criteria vague.** "It should work" is not a success criterion. "It should be fast" is not a success criterion. If the success criterion cannot be evaluated by someone other than its author without asking a follow-up question, rewrite it.

**Writing SCOPE.md after planning has already begun.** Retrofitting a scope document onto a plan that already exists is document theater. The plan will not change to match the scope. The scope will be written to match the plan. This defeats the purpose. SCOPE.md must come first.

**Allowing scope to drift silently.** When a task grows during execution — a natural occurrence — the SCOPE.md must be updated and re-confirmed. Silent scope expansion is the mechanism by which "quick features" become multi-week projects.

---

## Execution Steps

1. Before producing any plan, code, or architecture document, state: "I need to anchor scope before proceeding."
2. Ask or derive answers to the four mandatory questions. If the request does not contain enough information, ask — do not assume.
3. Write SCOPE.md using the format above.
4. Present SCOPE.md for confirmation. If operating autonomously, note any assumptions that require validation.
5. On confirmation, proceed with downstream work. Reference SCOPE.md in any plan or architecture document produced.
6. If scope changes during execution, update SCOPE.md first, then continue.

---

## Worked Example

**Request:** "Build a REST API for user notifications"

### Step 1 — State the gate

"I need to anchor scope before proceeding. Notification APIs have highly variable surface area — delivery channels, storage, user preferences, retry logic — and without pinning what's in and out we'll plan the wrong thing."

### Step 2 — Ask the four mandatory questions

**Q1 — Who is this for?**
> Is the primary consumer a frontend web client, a mobile app, another backend service, or all three? Who calls this API in production?

Answer received: "Frontend React app and our iOS client. No backend-to-backend calls."

**Q2 — What platform / stack?**
> What language and framework? Any constraints on the runtime or deployment environment?

Answer received: "Node 18, Express, deployed to Railway. PostgreSQL for storage."

**Q3 — In scope vs out of scope?**
> Should the API handle delivery (email, push, SMS), or only store and retrieve notification records for the frontend to render? Is user preference management (mute, frequency caps) in scope?

Answer received: "Store and retrieve only. The frontend renders them. Preferences are out of scope for now."

**Q4 — Success criteria?**
> What does done look like? Any performance requirements, existing test coverage thresholds, or schema contracts to honor?

Answer received: "An authenticated user can fetch their unread notifications and mark them read. Response time under 200ms at p95. Existing test suite must stay green."

### Step 3 — Write SCOPE.md

```markdown
# Scope: REST API for User Notifications

## Audience
Frontend React app and iOS client developers. No backend-to-backend consumers in this release.

## Platform and Stack
Node 18, Express, PostgreSQL. Deployed to Railway.

## Scope Boundaries

### In Scope
- GET /notifications — paginated list of notifications for the authenticated user
- PATCH /notifications/:id/read — mark a single notification as read
- POST /notifications/read-all — mark all unread notifications as read
- Database schema for notifications table (user_id, type, body, read_at, created_at)
- Authentication via existing session middleware (already present in codebase)

### Out of Scope
- Notification delivery (email, push, SMS) — not in this release
- User preference management (mute, frequency caps) — deferred to v2
- Admin endpoints for creating or broadcasting notifications
- Real-time delivery (WebSocket or SSE)
- Notification grouping or threading

## Success Criteria
- An authenticated user can retrieve their unread notifications via GET /notifications
- A user can mark one or all notifications as read
- p95 response time under 200ms under representative load
- All existing tests pass after implementation
- Response schema matches the contract in docs/api-schema.json (to be written as part of this task)

## Constraints and Assumptions
- Session middleware is already implemented — this task does not modify auth logic
- PostgreSQL schema changes will be handled via the existing migration tool (knex)
- Assuming read access to the users table via foreign key; no cross-schema queries
- No soft-delete or notification expiry in scope — records are permanent for now
```

### Step 4 — Confirm and gate

"Here is the SCOPE.md. Please confirm this accurately represents the task, or call out anything that's wrong or missing. Once confirmed, I'll proceed to PLAN.md."

---

## References

- [Scope anti-patterns](references/scope-anti-patterns.md) — named failure modes, detection signals, and recovery actions
