# Scope Anti-Patterns

Named failure modes in scope definition, with detection signals and recovery actions. Each pattern represents a recurring way that scope ambiguity enters a task and degrades the quality of AI-generated outputs.

---

## 1. Implied Audience

**Symptom:** SCOPE.md or the original request refers to "users", "the team", "developers", or "people" without further qualification. The consumer of the output is never named.

**Why it causes failures:** Different audiences require different interfaces, different levels of abstraction, and different success criteria. A CLI built for a DevOps engineer running it in a pipeline has entirely different UX requirements than one built for a developer running it interactively. Building for an implied audience means building for a phantom — usually the author's own mental model.

**Detection signal:** Ask "who specifically will use this?" If the answer takes more than one sentence to specify, the audience was implied.

**Recovery action:** Name the primary audience by role, context, and technical level. If there are multiple audiences, rank them and make trade-offs explicit. Rewrite any success criteria that assumed the phantom audience.

---

## 2. Scope Creep by Analogy

**Symptom:** Mid-execution additions prefixed by "while we're here", "it would also make sense to", "while we're touching this file", or "this is basically the same thing".

**Why it causes failures:** Analogical scope additions feel justified in the moment because they share surface-area with the original task. They are not the same task. Each addition shifts the success criteria, increases the blast radius of changes, and dilutes the definition of done. In AI-assisted workflows, scope creep by analogy is how a focused one-hour task becomes a four-hour task that touches twelve files.

**Detection signal:** Any addition that was not in the original SCOPE.md in-scope list. The test is not whether it is related — it is whether it was agreed to.

**Recovery action:** Stop. Log the addition to a backlog or a new scope-anchor session. Complete the original task first. Additions that are truly urgent get their own SCOPE.md and their own plan.

---

## 3. Success Theater

**Symptom:** Success criteria use adjectives without thresholds: "performant", "clean", "easy to use", "production-ready", "robust", "maintainable". The criteria cannot be evaluated without asking the author what they meant.

**Why it causes failures:** When success criteria are unmeasurable, any output can be declared done. The output satisfies the criteria because the criteria are compatible with anything. This creates the illusion of a completed task while the actual requirement — whatever the requester meant by "production-ready" — remains unmet and unstated.

**Detection signal:** Read each success criterion and ask: "Could someone other than the author evaluate this without a follow-up question?" If no, it is theater.

**Recovery action:** Rewrite each criterion as an observable outcome. Replace adjectives with thresholds or examples. "Production-ready" becomes "handles errors without crashing, logs structured JSON to stdout, and has at least one integration test covering the happy path."

---

## 4. Platform Assumption

**Symptom:** Code, architecture, or plans are designed for a specific runtime, OS, or framework that was never stated in the scope. The assumption surfaces late — usually during testing or deployment.

**Why it causes failures:** Platform assumptions are invisible to the builder. Code written for Node 18 may not run on Node 20. A shell script written for bash may not run on zsh. An architecture designed for a stateful server may not fit a serverless deployment target. The larger the platform gap, the more expensive the rework.

**Detection signal:** Review the plan or code and ask: "Is any platform-specific behavior being assumed that is not stated in SCOPE.md?" Common signals include hardcoded paths, OS-specific commands, version-specific API usage, or framework idioms that don't exist in other frameworks.

**Recovery action:** Add a Platform and Stack section to SCOPE.md if absent. State the runtime, OS, and version constraints explicitly. If multi-platform support is required, state it in scope and add a corresponding success criterion.

---

## 5. Boundary Blur

**Symptom:** SCOPE.md has an in-scope list but no out-of-scope list. Or the out-of-scope list is empty. Everything adjacent to the task is potentially in scope.

**Why it causes failures:** Without an explicit boundary, scope expands to fill available time and context. In AI-assisted workflows, boundary blur causes the model to speculatively include adjacent functionality — authentication when only authorization was requested, logging infrastructure when only a log line was needed, a full configuration system when only one environment variable was required. The output is overbuilt and the original requirement is buried.

**Detection signal:** Review the out-of-scope list. If it is empty or absent, boundary blur is present. The out-of-scope list should always contain at least two or three items — things that are adjacent, tempting, or commonly assumed.

**Recovery action:** Enumerate adjacent concerns explicitly and assign each one to in-scope or out-of-scope. Pay particular attention to cross-cutting concerns: error handling, logging, configuration, testing, documentation, and monitoring. State explicitly which of these are in scope for this task.

---

## 6. Moving Target

**Symptom:** SCOPE.md exists but is not updated when requirements change. The plan or code diverges from SCOPE.md silently. By the end of execution, SCOPE.md describes a different task than the one that was built.

**Why it causes failures:** A SCOPE.md that no longer matches the work being done is worse than no SCOPE.md. It creates false confidence. Reviewers and downstream agents trust SCOPE.md to describe what was built. When it does not, audits fail, handoffs break, and success criteria are evaluated against the wrong deliverable.

**Detection signal:** Compare SCOPE.md against the current plan or implementation. If they diverge — different success criteria, different in-scope items, different audience — the target has moved.

**Recovery action:** Stop execution. Update SCOPE.md to reflect the current state of the task. If the change is significant, re-confirm SCOPE.md before continuing. Treat the update as a formal scope change, not a housekeeping edit.

---

## 7. Capability Conflation

**Symptom:** SCOPE.md mixes what to build with how to build it. Architectural choices, technology selections, or implementation patterns appear in the scope definition rather than in the plan.

**Why it causes failures:** Scope and architecture serve different purposes. Scope defines the boundary of the problem. Architecture defines the solution approach. When they are conflated, architectural decisions become locked in before the problem is fully understood. Changing scope requires revisiting architecture, and vice versa. The documents become mutually inconsistent, and the task loses a stable reference point.

**Detection signal:** SCOPE.md contains sentences like "we will use Redis for caching", "the implementation will follow the repository pattern", or "this will be a React component". These are architectural decisions, not scope statements.

**Recovery action:** Move all how statements out of SCOPE.md and into PLAN.md or an ADR. SCOPE.md answers what is being built and for whom. If a technology choice is a constraint (e.g., "must use the existing PostgreSQL instance"), state it in the Constraints and Assumptions section, not the scope boundary.

---

## 8. Goldilocks Scope

**Symptom:** Scope is either so broad it describes a multi-month roadmap, or so narrow it describes a single line change that needs no scope document at all.

**Why it causes failures (too broad):** A scope that covers an entire product surface area cannot be planned, executed, or verified in a single session. Planning against a roadmap-sized scope produces plans that are too abstract to act on. Success criteria become unmeasurable. The task never finishes because "done" requires finishing the entire roadmap.

**Why it causes failures (too narrow):** Scope documents that describe trivially contained changes waste time and create process overhead with no benefit. They also desensitize teams to scope documents, reducing the signal value of SCOPE.md when it actually matters.

**Detection signal (too broad):** The in-scope list has more than eight to ten items, or success criteria reference more than one major functional area. The task cannot be completed in a single planning and execution cycle.

**Detection signal (too narrow):** The entire task fits in one function, one configuration value, or one file with no cross-cutting concerns. The reproduction case or change request already fully defines the scope.

**Recovery action (too broad):** Decompose the scope into phases or milestones. Each phase gets its own SCOPE.md. The current SCOPE.md covers only the immediate phase, with a note pointing to the larger roadmap.

**Recovery action (too narrow):** Skip the scope-anchor invocation. Apply the change directly. If scope expands during execution, invoke scope-anchor at that point.
