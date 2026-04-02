# Charter Anti-Patterns

Six named failure modes that appear repeatedly in PROJECT.md drafts. Each entry names the pattern, describes how to diagnose it, explains the downstream cost, and provides a repair strategy.

---

## 1. Feature Masquerade

**What it looks like**

The problem statement describes a solution rather than a problem. Phrases like "we need a tool that...", "the system should...", or "users will be able to..." appear in the first paragraph.

**Diagnosis**

Read the problem statement and ask: does this paragraph describe a state of the world that is bad, or does it describe something being built? If a non-technical stakeholder could read it and not realize they are being asked to approve a specific implementation, it is a Feature Masquerade.

Example of the pattern:
> "We need a REST API with JWT authentication that exposes time-tracking data to mobile clients."

This describes an implementation. The problem is invisible.

Repaired:
> "Mobile field teams cannot access project time data without VPN access to internal tools, causing delays in client billing that average 4 days past the work period."

**Downstream cost**

Features built to satisfy a Feature Masquerade problem statement are never validated against real user need. The team ships the described solution and then discovers it does not solve the underlying problem. Discovery of the real problem at that point requires rework proportional to the entire build.

**Repair**

Apply the five-whys to the stated solution until the actual human problem surfaces. Rewrite the problem statement starting from that human problem, with no implementation language.

---

## 2. Audience Soup

**What it looks like**

The target audience section lists three or more equal audiences with no primary designation. Each audience has distinct needs, and the section makes no priority call.

**Diagnosis**

Count the audiences listed. If there are more than two, or if no audience is labeled "primary," ask: whose unmet need is the project primarily solving? If the answer is "all of them equally," the project scope is not defined.

Example of the pattern:
> "Target users include developers, engineering managers, finance teams, and external clients."

No primary audience is identified. These four groups have fundamentally different goals. A system designed for all of them simultaneously is designed for none of them specifically.

**Downstream cost**

Without a primary audience, SPEC.md use cases have no consistent actor. Feature prioritization becomes political rather than principled. Architecture decisions made to serve managers conflict with usability decisions made to serve developers. The product ships with incoherent UX optimized for no one.

**Repair**

Force a single primary audience designation. Ask: if we could only delight one group, which group's delight would cause the others to adopt the product anyway? That group is primary. List others as secondary with explicit notes on where their needs are subordinate.

---

## 3. Vanity Metric

**What it looks like**

Success metrics measure activity or adoption rather than outcomes. Common vanity metrics: number of users, number of sessions, number of features shipped, stars on GitHub.

**Diagnosis**

Ask of each metric: could this number increase while the project's core problem remains unsolved? If yes, it is a vanity metric.

Example of the pattern:
> "Success: 500 registered users in the first 90 days."

Five hundred users who cannot accomplish the core task are not a success signal. This metric measures acquisition, not value delivery.

Repaired:
> "90% of registered users complete at least one full timesheet submission within their first week."

**Downstream cost**

Teams optimize for vanity metrics because they are easier to move. The product accumulates growth features (referral programs, onboarding funnels) while the core workflow remains broken. When the vanity metric is hit and stakeholders ask why the business outcome did not follow, the answer is always that the wrong thing was measured.

**Repair**

For each metric, identify the downstream behavior that would prove the problem is solved. Rewrite the metric to measure that behavior. If the behavior cannot be measured with available instrumentation, add instrumentation as a project requirement.

---

## 4. Deadline Theater

**What it looks like**

The timeline section contains a single end date with no intermediate milestones. Alternatively, it contains dates but no definition of what constitutes each milestone.

**Diagnosis**

Look at the timeline section. Count the milestones with observable definitions. If the count is zero, or if all milestones are phrased as "X complete" without specifying what observable event marks completion, it is Deadline Theater.

Example of the pattern:
> "Phase 1: January. Phase 2: February. Launch: March 15."

Nothing in this timeline is measurable until March 15. There is no early warning system for schedule risk.

Repaired:
> "Phase 1 milestone (January 31): CLI accepts a time entry and retrieves it correctly in five consecutive test runs on a clean machine.
> Phase 2 milestone (February 28): Entry created via CLI appears in target Jira ticket within 30 seconds on a live integration environment.
> Launch milestone (March 14): Ten pilot users each submit at least one week's timesheets via CLI with no support intervention."

**Downstream cost**

Deadline Theater conceals schedule risk until it is too late to recover. Without milestones, the first signal of failure is the missed launch date. With milestones, missed intermediate events give weeks of warning and time to course-correct.

**Repair**

Replace each phase boundary with a milestone statement that starts with a measurable observable event. If the team cannot define what done looks like for Phase 1, Phase 1 is not scoped — stop and scope it before continuing.

---

## 5. Constraint Omission

**What it looks like**

The constraints section is empty, says "none known," or lists only aspirational preferences rather than real constraints.

**Diagnosis**

Real constraints exist on every project. Ask four questions: What stack or infrastructure is mandated or excluded? What is the actual team size and availability? Are there compliance, legal, or data residency requirements? What must this system never do or never be? If the constraint section does not answer at least three of these, it is incomplete.

Example of the pattern:
> "Constraints: None at this time. We have flexibility on tech stack and deployment."

This is almost never true. It means the constraints were not investigated.

**Downstream cost**

Constraints discovered mid-architecture require rework at the most expensive possible moment. A database choice that violates a data residency requirement discovered in week six costs weeks of re-architecture. A compliance requirement discovered after launch costs audit fees and potential legal exposure.

**Repair**

Schedule a 30-minute constraint elicitation session with the technical lead, a stakeholder with budget authority, and (if applicable) a legal or compliance representative. Use the four questions above as the agenda. Record every constraint as confirmed or assumed. Flag assumed constraints for follow-up before ARCH.md begins.

---

## 6. Scope Leakage

**What it looks like**

PROJECT.md contains design decisions that belong in ARCH.md, SPEC.md, or DEV-SPEC.md. Database choices, framework selections, API designs, deployment topology, or UI wireframe descriptions appear in the charter.

**Diagnosis**

Read each sentence in PROJECT.md and ask: is this describing the problem, who has it, how success is measured, when milestones occur, or what limits the solution space? If a sentence answers "what will be built" or "how it will be built," it has leaked past the charter boundary.

Example of the pattern:
> "The system will use PostgreSQL for persistence with Redis for session caching. The CLI will be built in Go using the Cobra framework. The Jira integration will use OAuth 2.0 with a 15-minute token refresh interval."

These are ARCH.md and DEV-SPEC.md decisions. They do not belong in PROJECT.md.

**Downstream cost**

Design decisions embedded in PROJECT.md get confirmed as part of the charter review and then become politically difficult to revisit during architecture. Teams that discover a better approach during ARCH.md face the friction of "changing what was already agreed." Scope leakage locks in decisions before the investigation that justifies them.

**Repair**

Move all implementation-level decisions out of PROJECT.md into a "Design Notes" scratch file. Flag them as candidates for ARCH.md once requirements are confirmed. PROJECT.md should read identically whether the solution is a CLI, a web app, or a mobile app — if it reads differently for each, it contains scope leakage.
