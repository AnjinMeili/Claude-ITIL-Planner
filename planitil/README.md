# planitil

End-to-end project planning and quality-gate system for Claude Code. A 12-skill, 6-agent pipeline that produces a complete, file-persisted artifact chain — from project charter through production release runbook — with quality gates at every phase transition.

Self-contained. No plugin dependencies required.

---

## Artifact Chain

```
PROJECT.md  →  SCOPE.md  →  SPEC.md  →  ARCH.md + ADR-NNN.md  →  DEV-SPEC.md  →  QA.md  →  RELEASE.md
```

All artifacts are written to `.planning/` at the project root. Memory registers (`DECISIONS.md`, `ASSUMPTIONS.md`, `RISKS.md`, `DEBT.md`) are also kept in `.planning/`.

---

## Skills

### Layer 1 — Information Architecture

| Skill | Produces | Description |
|---|---|---|
| `project-charter` | `PROJECT.md` | Captures problem statement, target audience, and success metrics |
| `scope-anchor` | `SCOPE.md` | Defines in-scope / out-of-scope boundary before design begins |
| `requirements-spec` | `SPEC.md` | Derives use cases, acceptance criteria, and NFRs from PROJECT.md |
| `arch-design` | `ARCH.md`, `ADR-NNN.md` | System context, component diagram, data model, interface contracts |
| `dev-discipline` | `DEV-SPEC.md` | Coding standards, review checklist, merge criteria |
| `qa-strategy` | `QA.md` | Test plan, coverage targets, acceptance criteria |
| `release-protocol` | `RELEASE.md` | Deployment runbook, approval gates, rollback procedure, monitoring window |

### Layer 2 — Quality Gates

| Skill | Role | Description |
|---|---|---|
| `failure-modes` | Pre-design / pre-plan | Catalogs failure modes before architecture or planning begins |
| `confidence-gate` | Pre-execution | Scores confidence on key assumptions before committing to execution |
| `plan-critic` | Plan → execution gate | Evaluator-optimizer loop; blocks execution until plan scores ≥ 8/10 |
| `prereq-chain` | Phase transitions | Enforces artifact prerequisites before advancing to the next phase |
| `scope-anchor` | Post-charter | Anchors scope before requirements begin (also Layer 1) |

### Layer 3 — Memory

| Skill | Produces | Description |
|---|---|---|
| `context-keeper` | `DECISIONS.md`, `ASSUMPTIONS.md`, `RISKS.md`, `DEBT.md` | Living registers that survive session resets |

---

## Agents

| Agent | Model | Tools | Role |
|---|---|---|---|
| `dev-orchestrator` | sonnet | Read, Agent, Write | Routes tasks to skills; enforces layer sequencing; initializes `.planning/` |
| `requirements-analyst` | sonnet | Read, Write, WebSearch | Produces SPEC.md from PROJECT.md |
| `architect` | sonnet | Read, Write, WebFetch | Produces ARCH.md and ADR files from SPEC.md |
| `code-reviewer` | sonnet | Read, Grep, Glob, Write | Reviews changed files; writes review to `.planning/reviews/` |
| `plan-critic-evaluator` | haiku | Read, Write | Scores plans against the 5-dimension rubric; writes scorecard to disk |
| `release-manager` | haiku | Read, Write | Executes RELEASE.md runbook step-by-step with human-in-the-loop gates |

---

## Installation

Copy this directory into your Claude Code plugins location:

```bash
cp -r planitil/ ~/.claude/plugins/planitil/
```

Restart Claude Code. The 12 skills will be available as slash commands and the 6 agents will be available for spawning.

---

## Usage

### Full pipeline (via dev-orchestrator)

```
/dev-orchestrator
```

The orchestrator will guide you through the complete pipeline from project-charter to release-protocol, enforcing prerequisite chain and quality gates at each transition.

### Individual skills

```
/project-charter          # Start here for a new project
/scope-anchor             # Lock scope before requirements
/requirements-spec        # Generate SPEC.md
/arch-design              # Generate ARCH.md + ADRs
/dev-discipline           # Generate DEV-SPEC.md
/qa-strategy              # Generate QA.md
/release-protocol         # Generate RELEASE.md
/context-keeper           # Record a decision, assumption, risk, or debt item
/plan-critic [plan-file]  # Score a plan before executing it
/failure-modes            # Catalog failure modes before committing to a design
/confidence-gate          # Check confidence before execution
/prereq-chain             # Verify phase prerequisites
```

### Walking a release

Spawn the `release-manager` agent and it will read `.planning/RELEASE.md`, present the deployment summary, and walk through each runbook step and approval gate one at a time.

---

## Design Principles

**File persistence first.** Every artifact is written to `.planning/` immediately after production. No skill returns output only as text.

**Quality gates are not optional.** `plan-critic` blocks execution until a plan scores ≥ 8/10. `prereq-chain` blocks phase advancement until upstream artifacts exist. `confidence-gate` surfaces unvalidated assumptions before they become production incidents.

**Self-contained.** planitil has no `requires:` dependencies on other plugins. All 12 skills and 6 agents are included.

**Network access is scoped.** Agents that use WebFetch or WebSearch are restricted to URLs referenced in project artifacts or domain terminology directly relevant to requirements. No open-ended research.

**No re-deployment without version increment.** `release-manager` detects a completed deployment record and blocks re-execution against the same version.

---

## Artifact Locations

| Artifact | Path |
|---|---|
| PROJECT.md | `.planning/PROJECT.md` |
| SCOPE.md | `.planning/SCOPE.md` |
| SPEC.md | `.planning/SPEC.md` |
| ARCH.md | `.planning/ARCH.md` |
| ADR-NNN.md | `.planning/ADR-NNN.md` |
| DEV-SPEC.md | `.planning/DEV-SPEC.md` |
| QA.md | `.planning/QA.md` |
| RELEASE.md | `.planning/RELEASE.md` |
| DECISIONS.md | `.planning/DECISIONS.md` |
| ASSUMPTIONS.md | `.planning/ASSUMPTIONS.md` |
| RISKS.md | `.planning/RISKS.md` |
| DEBT.md | `.planning/DEBT.md` |
| Code reviews | `.planning/reviews/REVIEW-[date]-[scope].md` |
| Plan scorecards | `.planning/[phase]/SCORECARD-[date].md` |

---

## License

Apache 2.0. See [LICENSE](LICENSE).
