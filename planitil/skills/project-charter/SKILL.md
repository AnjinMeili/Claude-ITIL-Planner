---
name: project-charter
description: |
  Use when starting a new project, application, system, tool, or service — before any scoping, architecture, or planning begins. Produces PROJECT.md, the root artifact that all downstream work references.

  <example>
  User: "I want to build a developer time-tracking CLI"
  Action: Run project-charter to capture the problem statement, target audience, success metrics, timeline, and constraints before any feature discussion or design begins.
  </example>

  <example>
  User: "Let's kick off the new API gateway initiative"
  Action: Run project-charter to establish the authoritative PROJECT.md so all subsequent SCOPE.md, SPEC.md, and ARCH.md work has a confirmed foundation to reference.
  </example>
version: 1.0.0
---

# project-charter

Produce PROJECT.md — the root artifact anchoring all downstream work. Every SCOPE.md, SPEC.md, ARCH.md, and DEV-SPEC.md traces back to this document. Start nothing else until PROJECT.md is confirmed.

## What This Skill Produces

A single PROJECT.md file with five sections:

1. **Problem Statement** — one paragraph, problem only
2. **Target Audience** — named primary user, secondary users, their goals
3. **Success Metrics** — 3–5 measurable outcomes
4. **Timeline and Milestones** — phases with observable milestone events
5. **Constraints** — technical, resource, regulatory, and non-negotiable requirements

## The Five Sections

### Problem Statement

Write one paragraph. State the problem being solved, who experiences it, and why current solutions are insufficient. Use zero solution language — no mention of features, interfaces, architectures, or implementation choices.

Weak: "We need a CLI tool that tracks time automatically and syncs to Jira."
Strong: "Developers on distributed teams lose billable hours because manual time entry is deferred, forgotten, or rounded, and existing tools require context-switching out of the terminal — the environment where work actually happens."

The problem statement sets the scope boundary. Any work not traceable to this paragraph does not belong in this project.

### Target Audience

Name the primary user specifically. "Developers" is not specific. "Backend engineers working in terminal-first environments on billable client projects" is specific. The primary audience is singular — one archetype whose needs, when satisfied, define a successful product.

Secondary audiences are listed but subordinate. Their needs do not override the primary. If two audiences have conflicting needs, the primary wins.

For each audience entry, state what they are trying to accomplish — not what the system will give them.

### Success Metrics

List 3–5 measurable outcomes. Outcomes describe a state of the world that is verifiably true or false. Features are not outcomes.

Format each metric as a falsifiable statement:
- "Engineers can submit a complete timesheet in under 30 seconds without leaving the terminal" — measurable
- "Time entry error rate drops below 5% compared to the current manual baseline" — measurable
- "90% of users complete onboarding without consulting documentation" — measurable
- "Has time-tracking features" — not a metric, do not write this

Every success metric must be testable before release. If it cannot be measured, it cannot be confirmed. Rewrite or drop it.

### Timeline and Milestones

Divide work into phases with target date ranges. Each phase ends at a milestone — an observable event, not a task completion.

Observable milestone: "First end-to-end time entry stored and retrieved from local database"
Non-observable milestone: "Backend development complete"

Milestones are checkpoints that answer: "Would an external observer agree this has happened?" If the answer requires reading source code, it is not a milestone.

Include a final milestone that maps to the success metrics. The project is not done when features ship — it is done when outcomes are confirmed.

### Constraints

Constraints exist on every project. Listing none means they were not investigated, not that there are none.

Categories to assess:

**Technical constraints** — required stack, deployment environment, integration points, performance floors, data residency requirements.

**Resource constraints** — team size, availability, budget ceiling, external dependencies with fixed timelines.

**Regulatory constraints** — compliance requirements, audit trails, data handling obligations.

**Non-negotiables** — requirements that cannot be traded against scope or schedule. State them plainly. "Must run on-premise" is a non-negotiable constraint, not a preference.

If a constraint is assumed rather than confirmed, flag it: "(assumed — confirm with stakeholder)".

## The Gate Rule

Do not start SCOPE.md, SPEC.md, PLAN.md, or any design artifact until PROJECT.md has been reviewed and confirmed by the initiating party. The confirmation can be a simple "looks right, proceed" — but it must happen.

This gate exists because every downstream artifact inherits the framing of PROJECT.md. Errors in problem framing compound. An architecture built on a misunderstood problem statement wastes more time than the thirty minutes spent confirming the charter.

## Worked Example

**Trigger:** User says "build a developer time-tracking CLI."

**Resulting PROJECT.md (condensed):**

---

**Problem Statement**

Developers on billable projects lose revenue and waste time on end-of-week time reconstruction. Manual entry tools require leaving the terminal, breaking flow. Context-switching to web dashboards introduces delays long enough that accurate timestamps are forgotten. No existing tool integrates directly into the shell workflow where development work occurs.

**Target Audience**

Primary: Freelance and agency backend engineers who bill by the hour, work primarily in terminal environments, and use project-management systems (Jira, Linear, GitHub Issues) as the source of truth for work units.

Secondary: Engineering managers who need weekly utilization reports without chasing team members.

**Success Metrics**

- Engineers submit a complete day's timesheet in under 60 seconds from the terminal
- Time entries are accurate to ±5 minutes compared to actual work sessions, verified by post-hoc calendar comparison
- 85% of users complete initial setup (auth, project config) in one session without support
- Integration with at least one PM system (Jira or Linear) requires zero manual copy-paste

**Timeline and Milestones**

- Phase 1 (Weeks 1–3): Local storage and CLI entry — milestone: time entry created, stored, and retrieved via CLI with no external dependencies
- Phase 2 (Weeks 4–6): PM system integration — milestone: entry created in CLI syncs to Jira ticket, visible in Jira UI without manual action
- Phase 3 (Weeks 7–8): Reporting and onboarding — milestone: new user completes setup and submits first synced entry in under 10 minutes with no documentation

**Constraints**

- Must run on macOS and Linux; Windows is out of scope for v1
- No cloud infrastructure in v1; all data stored locally
- Must integrate with Jira Cloud API v3 (existing enterprise agreements in place)
- Team: 1 engineer, part-time; no designer
- Budget: zero; open-source dependencies only

---

## Common Mistakes

**Writing features instead of problems.** If the problem statement mentions a UI, an API, a database, or a CLI, it has slipped into solution space. Rewrite until only the problem and its human cost remain.

**Listing all stakeholders as equal audiences.** Every project has one primary audience. Identify it. Secondary audiences inform but do not drive design decisions.

**Treating milestones as task lists.** "Complete backend API" is a task. "API returns valid time entries for all test scenarios" is a milestone.

**Omitting constraints.** Constraints that are discovered in week four cost ten times more to accommodate than constraints listed in week one. Ask explicitly: what stack is mandated? What is the team size? What must this never do?

**Scope leakage.** PROJECT.md is not the place for database schema sketches, framework choices, or deployment topology. Those belong in ARCH.md. If a design decision appears in PROJECT.md, move it.

## Writing the Artifact

When the five sections are complete and confirmed, write the artifact to disk:

- **Path:** `.planning/PROJECT.md` at the project root
- If `.planning/` does not exist, it will have been created by `dev-orchestrator` before this skill runs
- Confirm with: `PROJECT.md written to .planning/PROJECT.md`

Do not present only the content as text. PROJECT.md must exist on disk for all downstream skills to function.

## References

- [charter-anti-patterns.md](references/charter-anti-patterns.md) — six named anti-patterns with diagnosis and repair guidance
- [PROJECT.md.template](references/assets/PROJECT.md.template) — fill-in-the-blank template for all five sections
