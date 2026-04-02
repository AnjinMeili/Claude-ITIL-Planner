# Use Case Patterns

Guide for writing well-formed use cases in SPEC.md. Covers the actor-action-outcome pattern, common anti-patterns, and five weak-to-strong rewrites.

---

## The Actor-Action-Outcome Pattern

Every use case follows this structure:

> As [actor], I need to [action] so that [outcome].

Each element has a specific meaning and scope:

### Actor

The actor is a specific human archetype or external system with agency. The actor comes directly from the Target Audience section of PROJECT.md. Copy the language — the primary audience description in PROJECT.md is the source of truth.

Acceptable actors:
- "a freelance backend engineer billing by the hour"
- "an engineering manager reviewing weekly utilization"
- "the Jira webhook consumer" (external system with agency)

Unacceptable actors:
- "the user" — too vague, which user?
- "the system" — systems do not have goals, humans do
- "developers" — which developers? doing what kind of work?
- "an administrator" — what kind of administrator? what context?

If the actor cannot be named precisely, the Target Audience section of PROJECT.md is incomplete. Stop and fix it before writing use cases.

### Action

The action is what the actor needs to accomplish, stated at the goal level. It is not a UI interaction, not a system behavior, and not an implementation step.

The action answers: "What does the actor need to do?" not "How does the system help them do it?"

Acceptable actions:
- "start tracking time on a task"
- "submit accumulated time entries to my billing system"
- "view total hours worked across a project this week"

Unacceptable actions:
- "click the Start button" — UI interaction, not a goal
- "call the /track/start endpoint" — implementation detail
- "have the system record the current timestamp" — system behavior, not actor goal

### Outcome

The outcome is the value the actor receives when the action is successfully completed. It answers "why does this matter?" and connects the use case to the success metrics in PROJECT.md.

Every outcome should map to at least one success metric. If a use case's outcome cannot be traced to any success metric, question whether the use case is in scope.

Acceptable outcomes:
- "so that time is recorded without interrupting my workflow"
- "so that client billing reflects actual work without manual copy-paste"
- "so that I can generate invoices without reconstructing my week from memory"

Unacceptable outcomes:
- "so that the system works correctly" — tautological, no value stated
- "so that the feature is available" — feature-as-outcome, not a user benefit
- "so that data is saved" — implementation outcome, not user value

---

## Anti-Patterns

### Anti-Pattern 1: Feature Use Case

**What it looks like**

The use case describes a feature the system has, not a goal an actor pursues.

Example:
> "The system shall provide a `track list` command that displays all time entries."

This is a feature specification, not a use case. There is no actor. There is no outcome. There is no "why."

**Repair**

Identify who would use this capability and what they would accomplish with it. Then write the use case from that actor's perspective.

Repaired:
> "As a freelance engineer, I need to review my recorded entries for the day so that I can verify completeness before submitting them to my client."

---

### Anti-Pattern 2: No Actor

**What it looks like**

The use case begins with an action or system behavior, skipping the actor entirely.

Example:
> "I need to authenticate before accessing time entries."

Who is "I"? This could be any user, an external system, or a hypothetical. Without a named actor, the use case cannot drive acceptance criteria with a specific Given state.

**Repair**

Name the actor from PROJECT.md's Target Audience section.

Repaired:
> "As a freelance backend engineer, I need to authenticate with my Jira credentials before accessing project time data so that only authorized project data is visible to me."

---

### Anti-Pattern 3: UI Use Case

**What it looks like**

The use case describes steps in a UI interaction flow rather than a user goal.

Example:
> "As a user, I need to open the settings menu, select 'Integrations', and enter my Jira API token so that I can save my credentials."

This describes a UI flow, not a goal. It locks the design into a specific interaction pattern before architecture is defined.

**Repair**

Collapse the UI steps into the goal they serve.

Repaired:
> "As a freelance engineer, I need to configure my Jira authentication credentials so that the CLI can submit time entries on my behalf without prompting me each time."

---

### Anti-Pattern 4: Compound Use Case

**What it looks like**

The use case contains multiple goals joined by "and."

Example:
> "As a billing engineer, I need to start a timer, switch tasks, and stop the timer at the end of the day so that my hours are recorded accurately."

This is three use cases compressed into one. Each goal has distinct acceptance criteria and potentially different failure paths.

**Repair**

Split into separate use cases, one per goal.

Repaired:
- UC-01: "As a billing engineer, I need to start a timer when beginning a task so that time is recorded from the moment work begins."
- UC-02: "As a billing engineer, I need to switch the active timer to a different task so that time is attributed to the correct work item without stopping and restarting manually."
- UC-03: "As a billing engineer, I need to stop the active timer at the end of a work session so that the final time entry is complete and ready for submission."

---

### Anti-Pattern 5: Outcome-Free Use Case

**What it looks like**

The use case has an actor and an action but the "so that" clause is missing or vacuous.

Example:
> "As a freelance engineer, I need to export my time entries."

Export to what format? For what purpose? Without the outcome, acceptance criteria cannot determine whether "export to CSV" or "export to PDF" or "export to Jira" satisfies the requirement.

**Repair**

Complete the outcome with the specific value the actor receives.

Repaired:
> "As a freelance engineer, I need to export my time entries as a CSV file so that I can import them into my invoicing tool without manual data entry."

---

## Weak-to-Strong Rewrites

### Rewrite 1

**Weak:** "Users should be able to log time."

**Problems:** "Users" is undefined. "Log time" describes a system capability, not a goal. No outcome.

**Strong:** "As a freelance backend engineer, I need to record time spent on a client task from my terminal so that billable hours are captured at the moment of work without requiring context-switching to a separate application."

---

### Rewrite 2

**Weak:** "The system needs to integrate with Jira."

**Problems:** "The system" is not an actor. "Integrate with Jira" is an implementation direction, not a goal. No actor, no action at goal level, no outcome.

**Strong:** "As a freelance backend engineer, I need my recorded time entries to appear on the corresponding Jira ticket so that project managers have accurate time data without requiring me to duplicate entry in two systems."

---

### Rewrite 3

**Weak:** "As a user, I need reports."

**Problems:** "User" is undefined. "Reports" is not an action — it is a feature category. No outcome.

**Strong:** "As an engineering manager, I need to view total hours logged per team member across a sprint so that I can identify workload imbalances before they cause deadline risk."

---

### Rewrite 4

**Weak:** "As a developer, I need to authenticate."

**Problems:** "Developer" is not specific enough. "Authenticate" describes a system mechanism, not a user goal. No outcome.

**Strong:** "As a freelance backend engineer setting up the CLI for the first time, I need to connect my Jira account once so that subsequent commands run without re-entering credentials."

---

### Rewrite 5

**Weak:** "The CLI should support multiple projects."

**Problems:** No actor. "Support multiple projects" is a feature scope statement. No outcome.

**Strong:** "As a freelance backend engineer working across multiple client engagements simultaneously, I need to switch the active project context without leaving my terminal so that time is attributed to the correct client without manual correction."
