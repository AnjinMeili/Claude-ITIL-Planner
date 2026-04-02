# Claude ITIL Planner

A structured planning and execution framework for Claude Code that enforces a complete software development lifecycle — from problem statement through production deployment — using a layered system of skills, agents, and persistent memory registers.

---

## What It Is

Claude ITIL Planner (planitil) is a Claude Code plugin that brings discipline to AI-assisted software development. It replaces ad-hoc planning with an enforced artifact chain, gates execution on measurable quality thresholds, and maintains institutional memory across session resets.

The system is built around a single principle: **work that cannot be traced to a confirmed requirement, evaluated against an explicit rubric, or recovered from a named failure mode should not proceed.**

---

## How It Works

Planitil operates in three layers. Each layer must be substantially complete before the next begins.

```
Layer 1: Information Architecture
  PROJECT.md → SCOPE.md → SPEC.md → ARCH.md → DEV-SPEC.md → QA.md → RELEASE.md

Layer 2: Quality Gates
  failure-modes → confidence-gate → plan-critic → prereq-chain
  (run throughout Layer 1; block advancement on failure)

Layer 3: Memory
  DECISIONS.md + ASSUMPTIONS.md + RISKS.md + DEBT.md
  (append-only registers; survive session resets; restore context in <5 minutes)
```

A `dev-orchestrator` agent routes tasks through this sequence, enforces prerequisites, and invokes the appropriate specialist skill. It detects missing artifacts and redirects rather than proceeding on assumptions.

---

## The Artifact Chain

Every artifact lives in `.planning/` and is a prerequisite gate for the next.

| Skill | Artifact | Purpose |
|---|---|---|
| `project-charter` | `PROJECT.md` | Problem statement, audience, success metrics, constraints |
| `scope-anchor` | `SCOPE.md` | Explicit in/out-of-scope list; success criteria |
| `requirements-spec` | `SPEC.md` | Use cases (UC-NN), acceptance criteria (AC-NN-MM), NFRs, dependencies |
| `arch-design` | `ARCH.md` + `ADR-NNN.md` | Component diagram, data model, interface contracts, deployment model |
| `dev-discipline` | `DEV-SPEC.md` | Code standards, commit protocol, review criteria, debt policy |
| `qa-strategy` | `QA.md` | Test matrix, UAT criteria, regression scenarios, observability spec |
| `release-protocol` | `RELEASE.md` | Deployment runbook, approval gates, rollback procedure, monitoring window |

Each artifact is confirmed before the next begins. No downstream skill runs against an unconfirmed upstream artifact.

---

## Skills

### Layer 1: Information Architecture

**`project-charter`** — Produces `PROJECT.md`. The root artifact. Captures the problem being solved in solution-free language: one-paragraph problem statement, named primary audience, 3–5 falsifiable success metrics, observable milestones, and explicit constraints. No downstream work starts until this is confirmed.

**`scope-anchor`** — Produces `SCOPE.md`. Answers four mandatory questions before any planning begins: who is this for, what platform/stack, what is explicitly in scope vs out of scope, and what does success look like. The out-of-scope list is as important as the in-scope list. A plan written without a SCOPE.md incurs an automatic -1 on the plan-critic Scope Fit dimension.

**`requirements-spec`** — Produces `SPEC.md` via the `requirements-analyst` agent. Derives 3–10 use cases in actor-action-outcome format (UC-01, UC-02…), 2–5 Given/When/Then acceptance criteria per use case (AC-01-01, AC-01-02…) with test type labels, non-functional requirements with explicit thresholds, and external dependency failure modes. Every element traces back to PROJECT.md.

**`arch-design`** — Produces `ARCH.md` and `ADR-NNN.md` files via the `architect` agent. ARCH.md covers system context, component diagram, data model, interface contracts, and deployment model. Each ADR records one significant decision with context, consequences (both easier and harder), and mandatory rejected alternatives. ADRs without rejected alternatives are not accepted.

**`dev-discipline`** — Produces `DEV-SPEC.md`. The working agreement for all contributors. Four sections: code standards (naming rules, file organization, type annotations, security rules), commit protocol (Conventional Commits with hook enforcement), review criteria (Critical/Important/Suggestion/Positive tier system), and debt management policy (three debt categories, DEBT.md format, triage cadence).

**`qa-strategy`** — Produces `QA.md`. Test matrix by component (type, tool, coverage target), UAT criteria derived from SPEC.md acceptance criteria (human-executable, no code knowledge required), regression scenarios for behaviors that must never break, and an observability specification (logging events, metrics, alerting thresholds).

**`release-protocol`** — Produces `RELEASE.md`. A deployment runbook where every step has a copy-paste command, measurable expected output, and a failure signal. Minimum two approval gates (QA pass + human sign-off). Rollback trigger is a specific measurable condition, not "if something goes wrong." Rollback procedure is tested before deployment begins.

### Layer 2: Quality Gates

**`failure-modes`** — Pre-execution checklist against 24 named failure patterns across three catalogs: planning (8 modes), architecture (8 modes), implementation (8 modes). Examples: Scope Inflation, Orphaned Goal, Single-Step Compression, Missing Rollback, Success Criteria Omission. Each pattern has an observable binary STOP condition. Work does not proceed if any STOP condition fires.

**`confidence-gate`** — Makes uncertainty visible before significant action. Scores confidence from 100, subtracting per uncertainty (High impact: -15, Medium: -8, Low: -3). Emits a visible `Confidence: XX%` block with categorized open uncertainties. Below 85%: ask 1–2 clarifying questions. Below 70%: hard stop, restate understanding, require confirmation before any action.

**`plan-critic`** — Evaluator-optimizer loop gating execution on plan quality. Scores plans across five dimensions (0–2 each, max 10): Goal Alignment, Completeness, Risk Surface, Scope Fit, Anti-Pattern Presence. Plans scoring below 8 receive specific revision instructions per failing dimension and are re-scored — up to two cycles before escalating to the user. Execution cannot proceed on a failing plan.

**`prereq-chain`** — Phase transition gate. Each phase has pre-defined completion criteria of four types: Artifact (file exists), Quality (measurable threshold), Decision (recorded with rationale), Confirmation (explicit acknowledgment). Emits `PHASE N → N+1 GATE: CLEARED` or `BLOCKED`. A BLOCKED signal is absolute — no proceed-with-caution override.

### Layer 3: Memory

**`context-keeper`** — Maintains four living registers in `.planning/`. Triggered automatically when decision undo cost exceeds 2 hours. Append-only; existing entries update only their Status field.

| Register | IDs | Contents |
|---|---|---|
| `DECISIONS.md` | DEC-NNN | What, why, alternatives considered, status |
| `ASSUMPTIONS.md` | ASS-NNN | Assumption, impact if wrong (H/M/L), validation method, status |
| `RISKS.md` | RISK-NNN | Risk, likelihood, impact, mitigation strategy, status |
| `DEBT.md` | DEBT-NNN | What deferred, why, impact, mandatory target resolution date |

Registers cross-link: decisions reference their risks, debts reference the causing decision. On session resume, dev-orchestrator reads all four and proposes the next logical step without requiring re-explanation.

---

## Agents

Skills spawn specialist agents for the work that requires them. Agents are not invoked directly by users in normal flow.

| Agent | Model | Spawned by | Role |
|---|---|---|---|
| `dev-orchestrator` | Sonnet | User | Routes tasks, enforces layer sequencing, blocks missing-prerequisite requests |
| `requirements-analyst` | Sonnet | `requirements-spec` | Reads PROJECT.md, derives use cases and ACs, writes SPEC.md |
| `architect` | Sonnet | `arch-design` | Reads PROJECT.md + SPEC.md, produces ARCH.md and ADRs |
| `code-reviewer` | Sonnet | Externally | Tiered review (Critical/Important/Suggestion/Positive) against DEV-SPEC.md |
| `plan-critic-evaluator` | Haiku | `plan-critic` | Scores plan against rubric; separate evaluator prevents self-review |
| `release-manager` | Haiku | `release-protocol` | Walks through RELEASE.md runbook step-by-step with human confirmation at each gate |

---

## What Makes It Different

### Named failure modes, not vague warnings
Twenty-four cataloged anti-patterns with binary STOP conditions. Before any planning, architecture, or implementation work, a checklist is presented. Work halts if a STOP condition fires — not warned, not noted, halted.

### Quality gates that actually block
`plan-critic` requires ≥8/10 before execution. `confidence-gate` requires ≥85% confidence or explicit clarification. `prereq-chain` BLOCKED signals are absolute. Every gate produces visible, specific output — not a hedged internal note.

### Evaluator separated from author
`plan-critic-evaluator` is a distinct Haiku agent from the skill that orchestrates the revision loop. The model that revised the plan is not the sole judge of its quality. This is the Anthropic evaluator-optimizer pattern applied at the planning level.

### Persistent memory that survives session resets
Four append-only registers (`DECISIONS.md`, `ASSUMPTIONS.md`, `RISKS.md`, `DEBT.md`) accumulate institutional memory. On session resume, `dev-orchestrator` reads all four and presents a summary of active decisions, unvalidated assumptions, open risks, and overdue debt — restoring context in under five minutes without re-explaining the project.

### Complete traceability in both directions
Every acceptance criterion traces to a use case. Every use case traces to PROJECT.md. QA test scenarios trace to acceptance criteria. PLAN.md tasks reference UC and AC IDs. Nothing exists without a parent; nothing is built without a requirement.

### Runbook-as-code
Every RELEASE.md step has a copy-paste command, a measurable expected output, and a failure signal. Approval gates require evidence artifacts (test output, review sign-off) — not subjective judgment. The rollback procedure is written and tested before the deployment begins.

### Debt requires a resolution date
Every DEBT.md entry must carry a target resolution date or milestone anchor. "Eventually" and "when we have time" are not accepted. Debt without a target is permanent.

---

## Quickstart

```
/planitil:project-charter    # Define the problem
/planitil:scope-anchor       # Anchor the boundary
/planitil:requirements-spec  # Derive use cases and acceptance criteria
/planitil:arch-design        # Design the system structure
/planitil:dev-discipline     # Establish working standards
/planitil:plan-critic        # Gate the implementation plan
/planitil:qa-strategy        # Define test and observability strategy
/planitil:release-protocol   # Write the deployment runbook
```

Or start with the orchestrator and let it route:

```
/planitil:dev-orchestrator   # Routes to the correct skill based on what exists in .planning/
```

---

## Directory Structure

```
planitil/
├── plugin.yaml                  # Plugin manifest
├── .claude-plugin/plugin.json   # Claude Code metadata
├── skills/                      # 12 skills
│   ├── project-charter/
│   ├── scope-anchor/
│   ├── requirements-spec/
│   ├── arch-design/
│   ├── dev-discipline/
│   ├── qa-strategy/
│   ├── release-protocol/
│   ├── failure-modes/
│   ├── confidence-gate/
│   ├── plan-critic/
│   ├── prereq-chain/
│   └── context-keeper/
├── agents/                      # 6 agent definitions
│   ├── dev-orchestrator.md
│   ├── requirements-analyst.md
│   ├── architect.md
│   ├── code-reviewer.md
│   ├── plan-critic-evaluator.md
│   └── release-manager.md
└── tests/                       # Test scenarios per skill
```

Each skill directory contains a `SKILL.md` (the skill prompt) and a `references/` directory with rubrics, guides, and templates that the skill and its agents reference during execution.

---

## Design Principles

- **Document what is already practiced, not impose a new regime.** `dev-discipline` on an existing project should capture the team's actual standards.
- **Completion criteria must be defined before phases start.** Criteria written after work is done match the work, not the requirement.
- **Alternatives rejected are mandatory.** A decision without rejected alternatives is an assumption.
- **STOP conditions are observable and binary.** Two people evaluating the same plan should reach the same conclusion.
- **Confirmation before proceeding.** Every foundational artifact requires explicit confirmation before downstream work begins.
- **Orchestration without simulation.** `dev-orchestrator` routes to skills; it does not simulate them. Missing skills are detected and reported.
