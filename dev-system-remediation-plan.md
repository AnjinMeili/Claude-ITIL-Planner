# dev-system Remediation Plan

**Source:** `dev-system-review.md` (2026-04-01)
**Updated:** 2026-04-01 — `workflow-quality` dependency audit completed (see §Dependency Resolution below)
**Goal:** Resolve all Critical and Important findings; address Suggestions where cost is low.
**Artifact chain:** This plan resolves 10 findings across 3 tiers (3 Critical, 4 Important, 3 Suggestion).

---

## Dependency Resolution: workflow-quality

`dev-system/plugin.yaml` declares:
```yaml
requires:
  - workflow-quality
```

`~/.claude/plugins/workflow-quality/` exists and contains all five Layer 2 skills referenced by `dev-orchestrator`:

| Skill referenced by dev-orchestrator | Present in workflow-quality? | Status |
|---|---|---|
| `scope-anchor` | ✅ Yes — full SKILL.md with worked example | **SATISFIED** |
| `failure-modes` | ✅ Yes — 3-catalog system (planning/architecture/implementation) | **SATISFIED** |
| `plan-critic` | ✅ Yes — evaluator-optimizer loop, spawns plan-critic-evaluator | **SATISFIED** |
| `prereq-chain` | ✅ Yes — phase gate protocol with CLEARED/BLOCKED output | **SATISFIED** |
| `confidence-gate` | ✅ Yes — uncertainty accounting with 85% threshold | **SATISFIED** |

**Impact on Phase 3:** Finding #4 (missing Layer 2 skills) is **structurally resolved** by the existing `workflow-quality` plugin. Tasks 3.2–3.6 (create new skill files) and Task 3.7 (update plugin.yaml) are **eliminated**. Only Task 3.1 (routing guard) is retained as a defensive hardening measure.

**Residual issue:** The `plan-critic` SKILL.md and `plan-critic-evaluator` agent both state the PASS threshold as `≥ 7.5`. This is consistent across both files (Finding #10 applies to both). The fix in Task 6.3 must be applied to **both** files, not just the agent.

---

## Phasing Overview

| Phase | Focus | Findings addressed | Risk |
|---|---|---|---|
| 1 — Security & Permissions | Stop unguarded execution | #1, #6 | Low — config edit only |
| 2 — Persistence gaps | Add Write to agents that need it | #2, #3 | Low — frontmatter edit only |
| 3 — Layer 2 routing guard | Harden orchestrator against missing skills | #4 | Low — workflow-quality satisfies the dependency; guard is defensive only |
| 4 — Skill write delegation | Ensure every skill can persist its artifact | #7 | Medium — each skill needs an agent or Write context |
| 5 — Path standardization | Lock artifact locations to `.planning/` | #8 | Medium — touches all skills + agents |
| 6 — Polish | Fix hardcoded paths, threshold, concurrency note | #5, #9, #10 | Low |

---

## Phase 1 — Security & Permissions

**Goal:** Restore permission prompts and constrain network-capable agents.
**Findings:** #1 (`settings.json`), #6 (`architect`, `requirements-analyst` network access)

### Task 1.1 — Revert `skipDangerousModePermissionPrompt`

**File:** `~/.claude/settings.json`
**Change:** Remove or set `"skipDangerousModePermissionPrompt": false`.
**Why:** With this flag active, all agent tool calls — including `WebFetch`, `WebSearch`, and `Write` — execute without user confirmation. This is the root amplifier for findings #2, #3, and #6.
**Acceptance:** Running any agent that calls `Write` presents a confirmation prompt before executing.

---

### Task 1.2 — Add network-use constraints to `architect` agent instructions

**File:** `~/.claude/agents/architect.md`
**Change:** Add a constraint section to the agent's instruction body:

```
## Network Access Policy

WebFetch is permitted only to retrieve publicly documented API specifications,
framework documentation, or standards documents directly relevant to the current
architecture decision. Do not fetch URLs that were not referenced in PROJECT.md
or SPEC.md. Log each URL fetched and the reason in the ARCH.md References section.
```

**Acceptance:** Agent instructions explicitly constrain the scope of WebFetch.

---

### Task 1.3 — Add network-use constraints to `requirements-analyst` agent instructions

**File:** `~/.claude/agents/requirements-analyst.md`
**Change:** Add equivalent constraint:

```
## Network Access Policy

WebSearch is permitted only to look up domain-specific terminology, regulatory
definitions, or industry standards that are directly relevant to requirements
being captured. Do not search for implementation approaches or technology choices
— those belong in arch-design. Log each search query and its purpose in the
SPEC.md External Dependencies section if it surfaces a new dependency.
```

**Acceptance:** Agent instructions explicitly constrain the scope of WebSearch.

---

## Phase 2 — Persistence Gaps (Critical Agents)

**Goal:** Ensure `code-reviewer` and `plan-critic-evaluator` write their output to disk.
**Findings:** #2 (code-reviewer), #3 (plan-critic-evaluator)

### Task 2.1 — Add Write tool to `code-reviewer` and define output artifact

**File:** `~/.claude/agents/code-reviewer.md`
**Change 1:** Update frontmatter tools line:
```
tools:
  - Read
  - Grep
  - Glob
  - Write
```

**Change 2:** Add output persistence instructions after the Review Output Format section:

```
## Persisting the Review

After producing the review, write it to disk:

- File path: `[project-root]/.planning/reviews/REVIEW-[YYYY-MM-DD]-[scope-slug].md`
  where `scope-slug` is the module name or PR identifier (lowercase, hyphens).
- If `.planning/reviews/` does not exist, create it.
- Confirm with: `Review written to [path].`

Do not return the review only as text. A review that is not written to disk
provides no persistent record and cannot be referenced by context-keeper.
```

**Acceptance:** After review, a `.md` file exists at the declared path containing the full review output.

---

### Task 2.2 — Add Write tool to `plan-critic-evaluator` and define output artifact

**File:** `~/.claude/agents/plan-critic-evaluator.md`
**Change 1:** Update frontmatter tools line:
```
tools:
  - Read
  - Write
```

**Change 2:** Add output persistence instructions after the Output Format section:

```
## Persisting the Scorecard

After producing the scorecard, write it to the same directory as the plan file:

- File path: `[plan-file-directory]/SCORECARD-[YYYY-MM-DD].md`
- If the plan file is `.planning/phases/02/PLAN.md`, write to `.planning/phases/02/SCORECARD-[date].md`.
- Confirm with: `Scorecard written to [path]. Verdict: PASS|FAIL`

Do not return the scorecard only as text.
```

**Acceptance:** After evaluation, a `SCORECARD-*.md` file exists adjacent to the evaluated PLAN.md.

---

## Phase 3 — Layer 2 Routing Guard

**Goal:** Harden `dev-orchestrator` against the edge case where `workflow-quality` is not installed, so routing failures are explicit rather than silent.
**Finding:** #4 (dev-orchestrator references Layer 2 skills)
**Status:** The root cause (missing skills) is **resolved** by `workflow-quality`. This phase is now purely defensive hardening.

> **Note:** `workflow-quality` provides all five Layer 2 skills (`scope-anchor`, `failure-modes`, `plan-critic`, `prereq-chain`, `confidence-gate`) and is already declared as a `requires` dependency in `dev-system/plugin.yaml`. Tasks 3.2–3.7 from the original plan are **eliminated**.

---

### Task 3.1 — Guard dev-orchestrator routing against missing skills *(defensive only)*

**File:** `~/.claude/agents/dev-orchestrator.md`
**Change:** Add a routing guard block after the Sequencing Rules section:

```
## Missing Skill Guard

Before routing to any Layer 2 skill, verify the skill is available.
If the target skill is not reachable, return:

ROUTING_ERROR: Skill `[skill-name]` is referenced in the routing table but is
not available. Ensure the workflow-quality plugin is installed. Layer 2 quality
gates cannot run without it.

Do not attempt to simulate a missing skill's behavior inline.
```

**Acceptance:** Routing to an unavailable Layer 2 skill produces a structured `ROUTING_ERROR`, not a silent LLM fallback.

---

## Phase 4 — Skill Write Delegation

**Goal:** Ensure every skill that declares it produces an artifact can actually write that artifact when invoked via `dev-orchestrator`.
**Finding:** #7 (skills have no tool declarations; dev-orchestrator has no Write tool)

The correct pattern — established by `arch-design` → `architect` — is: skill invokes a dedicated agent that has Write. Replicate this pattern for every skill that produces a file.

**Note:** Skills themselves cannot carry `tools:` frontmatter in the current Claude Code plugin model. The delegation pattern is the right architectural solution.

---

### Task 4.1 — Create `charter-writer` agent for `project-charter`

**File:** `~/.claude/agents/charter-writer.md`
**Tools:** `Read, Write`
**Purpose:** Receives the drafted PROJECT.md content from the project-charter skill and writes it to disk. Mirrors the `requirements-analyst` / `architect` pattern.
**Output:** `PROJECT.md` at project root.

---

### Task 4.2 — Update `project-charter` skill to spawn `charter-writer`

**File:** `~/.claude/plugins/dev-system/skills/project-charter/SKILL.md`
**Change:** Add a "Writing the Artifact" section that instructs the skill to spawn the `charter-writer` agent with the completed PROJECT.md content, matching the `arch-design` → `architect` delegation pattern.

---

### Task 4.3 — Create `spec-writer` agent for `requirements-spec`

**File:** `~/.claude/agents/spec-writer.md`
**Tools:** `Read, Write`
**Note:** This task may be redundant if `requirements-analyst` already covers this role. Audit first: if `requirements-spec` skill already spawns `requirements-analyst`, no new agent is needed — just verify the spawn instruction exists in the skill.

---

### Task 4.4 — Create `discipline-writer` agent for `dev-discipline`

**File:** `~/.claude/agents/discipline-writer.md`
**Tools:** `Read, Write`
**Output:** `DEV-SPEC.md` at project root.

---

### Task 4.5 — Create `qa-writer` agent for `qa-strategy`

**File:** `~/.claude/agents/qa-writer.md`
**Tools:** `Read, Write`
**Output:** `QA.md` at project root.

---

### Task 4.6 — Create `release-writer` agent for `release-protocol`

**File:** `~/.claude/agents/release-writer.md`
**Tools:** `Read, Write`
**Output:** `RELEASE.md` at project root.
**Note:** Distinct from `release-manager` — writer produces the document; manager executes it.

---

### Task 4.7 — Create `register-writer` agent for `context-keeper`

**File:** `~/.claude/agents/register-writer.md`
**Tools:** `Read, Write`
**Output:** `DECISIONS.md`, `ASSUMPTIONS.md`, `RISKS.md`, `DEBT.md` in `.planning/`.

---

## Phase 5 — Path Standardization

**Goal:** Lock all artifact output locations to `.planning/` with `dev-orchestrator` ensuring the directory exists at session start.
**Finding:** #8 (artifact locations inconsistent across skills and agents)

### Canonical location table (post-remediation)

| Artifact | Canonical path |
|---|---|
| `PROJECT.md` | `.planning/PROJECT.md` |
| `SCOPE.md` | `.planning/SCOPE.md` |
| `SPEC.md` | `.planning/SPEC.md` |
| `ARCH.md` | `.planning/ARCH.md` |
| `ADR-NNN.md` | `.planning/ADR-NNN.md` |
| `DEV-SPEC.md` | `.planning/DEV-SPEC.md` |
| `QA.md` | `.planning/QA.md` |
| `RELEASE.md` | `.planning/RELEASE.md` |
| `DECISIONS.md` | `.planning/DECISIONS.md` |
| `ASSUMPTIONS.md` | `.planning/ASSUMPTIONS.md` |
| `RISKS.md` | `.planning/RISKS.md` |
| `DEBT.md` | `.planning/DEBT.md` |
| `REVIEW-*.md` | `.planning/reviews/REVIEW-*.md` |
| `SCORECARD-*.md` | Adjacent to the evaluated `PLAN.md` |

**Exception:** `DEV-SPEC.md` may reasonably remain at project root since `code-reviewer` reads it from there. Accept either `.planning/DEV-SPEC.md` or project root as valid — document the convention in `dev-discipline`.

---

### Task 5.1 — Update `dev-orchestrator` to ensure `.planning/` exists at session start

**File:** `~/.claude/agents/dev-orchestrator.md`
**Change:** Add to the Mandatory First Actions section:

```
Before routing any task, verify that a `.planning/` directory exists at the
project root. If it does not exist, create it using the Write tool before
proceeding. This ensures all downstream artifact writes succeed.
```

**Note:** This requires adding `Write` to dev-orchestrator's tools list, which is a deliberate expansion of its permissions. Limit scope: `Write` is used only for directory initialization, not content generation.

---

### Task 5.2 — Update output paths in all skills and agents

For each component, update all references from "project root" or ".planning/ or project root" to the canonical path in the table above. Files to update:

- `skills/project-charter/SKILL.md`
- `skills/requirements-spec/SKILL.md`
- `skills/dev-discipline/SKILL.md`
- `skills/qa-strategy/SKILL.md`
- `skills/release-protocol/SKILL.md`
- `skills/context-keeper/SKILL.md`
- `agents/requirements-analyst.md`
- `agents/architect.md`
- `agents/release-manager.md`

---

## Phase 6 — Polish

**Goal:** Fix the three low-risk Suggestion findings.
**Findings:** #5 (hardcoded path), #9 (concurrency note), #10 (threshold clarification)

### Task 6.1 — Fix hardcoded absolute paths in `arch-design` SKILL.md

**File:** `~/.claude/plugins/dev-system/skills/arch-design/SKILL.md`
**Change:** Replace all three absolute-path references with relative paths:

| Before | After |
|---|---|
| `/Users/james/.claude/plugins/dev-system/skills/arch-design/references/adr-patterns.md` | `references/adr-patterns.md` |
| `/Users/james/.claude/plugins/dev-system/skills/arch-design/references/system-boundary-guide.md` | `references/system-boundary-guide.md` |
| `/Users/james/.claude/plugins/dev-system/skills/arch-design/references/assets/ADR.md.template` | `references/assets/ADR.md.template` |

---

### Task 6.2 — Add last-write-wins warning to `release-manager`

**File:** `~/.claude/agents/release-manager.md`
**Change:** Add a note in the Processing Approval Gates section:

```
Only one release-manager instance should be active per RELEASE.md at a time.
If you detect that RELEASE.md has been modified since you last read it (compare
the gate status fields you read at startup to what exists now), stop and return:
RELEASE_CONFLICT: RELEASE.md has been modified by another process. Re-read the
file before recording any further gate updates.
```

---

### Task 6.3 — Clarify PASS threshold in both `plan-critic` and `plan-critic-evaluator`

The 7.5 threshold appears in **two files** and must be updated consistently:

**File 1:** `~/.claude/agents/plan-critic-evaluator.md`
Replace:
```
- If Total >= 7.5: Verdict is PASS.
- If Total < 7.5: Verdict is FAIL.
```
With:
```
- If Total >= 8: Verdict is PASS.  (≥ 80% of 10 maximum points)
- If Total <= 7: Verdict is FAIL.
```

**File 2:** `~/.claude/plugins/workflow-quality/skills/plan-critic/SKILL.md`
Replace all three occurrences of `7.5` threshold language:
- "scores 7.5 or higher" → "scores 8 or higher (≥ 80% of 10 maximum points)"
- "Score ≥ 7.5: PASS" → "Score ≥ 8: PASS"
- "Passing threshold is 7.5" → "Passing threshold is 8 (≥ 80% of 10 maximum points)"

Also update the design rationale paragraph: *"A plan that is perfect on four dimensions and weak on one can still pass. A plan with two weak dimensions cannot."* — this remains accurate at threshold 8 and should be kept as the explanatory note.

**Why these changes are safe:** A score of 7 means two dimensions scored 1 (partial) — the current behavior intended by the 7.5 threshold was always to fail plans with two weak dimensions. Changing the written threshold from 7.5 to 8 aligns the text with actual behavior; no plans that previously passed will now fail, and no plans that previously failed will now pass.

---

## Milestone Summary

| Milestone | Phase complete | Observable signal |
|---|---|---|
| M1: No unguarded execution | Phase 1 done | Permission prompt appears on first agent Write call |
| M2: All agent outputs persist | Phase 2 done | REVIEW-*.md and SCORECARD-*.md files exist after agent runs |
| M3: Layer 2 stubs installed | Task 3.1 done | dev-orchestrator returns structured error for missing skills |
| M4: Layer 2 fully operational | Phase 3 done | Full three-layer pipeline executes end-to-end |
| M5: All skills persist artifacts | Phase 4 done | Every skill produces a .md file via its writer agent |
| M6: Unified artifact location | Phase 5 done | All artifacts land in `.planning/`; no "or project root" ambiguity |
| M7: All polish applied | Phase 6 done | Relative paths in arch-design; correct threshold; conflict guard |
| M8: planitil package assembled | Phase 7 done | `/Users/james/src/test/planitil/` installs cleanly; all compliance checks pass |

---

## Execution Order and Dependencies

```
Task 1.1 (settings.json) ─────────────────────────────────── no deps
Task 1.2 + 1.3 (network constraints) ─────────────────────── no deps

Task 2.1 (code-reviewer Write) ───────────────────────────── no deps
Task 2.2 (plan-critic Write) ─────────────────────────────── no deps

Task 3.1 (orchestrator guard) ────────────────────────────── no deps
  ↳ Tasks 3.2–3.7 ELIMINATED — satisfied by workflow-quality

Task 4.1–4.7 (writer agents) ─────────────────────────────── no deps
Tasks 4.2, 4.3 (skill updates) ──────── after writer agents ─

Task 5.1 (orchestrator .planning/) ── after Phase 4 ─────────
Task 5.2 (path updates) ─────────────── after 5.1 ──────────

Task 6.1 (arch-design paths) ────────────────────────────── no deps
Task 6.2 (release-manager guard) ────────────────────────── no deps
Task 6.3 (threshold — 2 files) ──────────────────────────── no deps
```

**Phases 1, 2, 3, and 6 can all start immediately and run in parallel.**
**Phase 4 can start immediately; writer agents are independent of each other.**
**Phase 5 depends on Phase 4 being complete.**
**Phase 7 depends on Phases 1–6 being complete** — it assembles the remediated versions of all artifacts.

---

## Files to Create (net new)

| File | Phase | Notes |
|---|---|---|
| `~/.claude/agents/charter-writer.md` | 4 | |
| `~/.claude/agents/discipline-writer.md` | 4 | |
| `~/.claude/agents/qa-writer.md` | 4 | |
| `~/.claude/agents/release-writer.md` | 4 | Distinct from release-manager |
| `~/.claude/agents/register-writer.md` | 4 | |
| ~~`~/.claude/plugins/dev-system/skills/scope-anchor/SKILL.md`~~ | ~~3~~ | **ELIMINATED** — exists in workflow-quality |
| ~~`~/.claude/plugins/dev-system/skills/failure-modes/SKILL.md`~~ | ~~3~~ | **ELIMINATED** — exists in workflow-quality |
| ~~`~/.claude/plugins/dev-system/skills/plan-critic/SKILL.md`~~ | ~~3~~ | **ELIMINATED** — exists in workflow-quality |
| ~~`~/.claude/plugins/dev-system/skills/prereq-chain/SKILL.md`~~ | ~~3~~ | **ELIMINATED** — exists in workflow-quality |
| ~~`~/.claude/plugins/dev-system/skills/confidence-gate/SKILL.md`~~ | ~~3~~ | **ELIMINATED** — exists in workflow-quality |

**Net new files required: 5** (down from 10)

## Files to Modify (existing)

| File | Phase | Tasks |
|---|---|---|
| `~/.claude/settings.json` | 1 | 1.1 |
| `~/.claude/agents/architect.md` | 1 | 1.2 |
| `~/.claude/agents/requirements-analyst.md` | 1 | 1.3 |
| `~/.claude/agents/code-reviewer.md` | 2 | 2.1 |
| `~/.claude/agents/plan-critic-evaluator.md` | 2, 6 | 2.2, 6.3 |
| `~/.claude/agents/dev-orchestrator.md` | 3, 5 | 3.1, 5.1 |
| `~/.claude/agents/release-manager.md` | 6 | 6.2 |
| ~~`~/.claude/plugins/dev-system/plugin.yaml`~~ | ~~3~~ | **ELIMINATED** — requires: workflow-quality already declared |
| `~/.claude/plugins/workflow-quality/skills/plan-critic/SKILL.md` | 6 | 6.3 — threshold fix |
| `~/.claude/plugins/dev-system/skills/arch-design/SKILL.md` | 5, 6 | 5.2, 6.1 |
| `~/.claude/plugins/dev-system/skills/project-charter/SKILL.md` | 4, 5 | 4.2, 5.2 |
| `~/.claude/plugins/dev-system/skills/requirements-spec/SKILL.md` | 4, 5 | 4.3, 5.2 |
| `~/.claude/plugins/dev-system/skills/dev-discipline/SKILL.md` | 4, 5 | 4.4, 5.2 |
| `~/.claude/plugins/dev-system/skills/qa-strategy/SKILL.md` | 4, 5 | 4.5, 5.2 |
| `~/.claude/plugins/dev-system/skills/release-protocol/SKILL.md` | 4, 5 | 4.6, 5.2 |
| `~/.claude/plugins/dev-system/skills/context-keeper/SKILL.md` | 4, 5 | 4.7, 5.2 |

---

## Phase 7 — Assemble `planitil` Installable Package

**Goal:** Combine the remediated `dev-system` and `workflow-quality` plugins into a single self-contained installable plugin named `planitil`, assembled at `/Users/james/src/test/planitil/`, conforming to Claude Code plugin best practices.

**Dependency:** Phases 1–6 must be complete. Phase 7 assembles the post-remediation versions of all artifacts — do not copy source files before patches from earlier phases are applied.

**Why merge rather than keep separate:** `dev-system` already declares `requires: workflow-quality`. A consumer installing `dev-system` must also install `workflow-quality` and maintain version alignment between them. `planitil` collapses this into a single installable unit with no external dependencies, one `plugin.json`, one README, and one coherent skill namespace.

---

### Target Directory Structure

```
/Users/james/src/test/planitil/
├── .claude-plugin/
│   └── plugin.json                          # canonical metadata (Task 7.1)
├── LICENSE                                  # Apache 2.0 (Task 7.2)
├── README.md                                # full feature documentation (Task 7.3)
├── plugin.yaml                              # extended metadata (Task 7.4)
│
├── skills/                                  # 12 skills — 7 from dev-system, 5 from workflow-quality
│   ├── project-charter/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── assets/PROJECT.md.template
│   │       └── charter-anti-patterns.md
│   ├── requirements-spec/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── assets/SPEC.md.template
│   │       ├── acceptance-criteria-guide.md
│   │       └── use-case-patterns.md
│   ├── arch-design/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── assets/ADR.md.template
│   │       ├── adr-patterns.md
│   │       └── system-boundary-guide.md
│   ├── dev-discipline/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── commit-protocol.md
│   │       ├── debt-management.md
│   │       └── review-criteria.md
│   ├── qa-strategy/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── observability-spec.md
│   │       ├── test-matrix-patterns.md
│   │       └── uat-criteria-guide.md
│   ├── release-protocol/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── assets/RELEASE.md.template
│   │       ├── rollback-strategies.md
│   │       └── runbook-patterns.md
│   ├── context-keeper/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── assumption-register.md
│   │       ├── decision-register.md
│   │       └── risk-debt-patterns.md
│   ├── scope-anchor/
│   │   ├── SKILL.md
│   │   └── references/
│   │       └── scope-anti-patterns.md
│   ├── failure-modes/
│   │   ├── SKILL.md
│   │   └── references/
│   │       └── failure-modes/
│   │           ├── architecture.md
│   │           ├── implementation.md
│   │           └── planning.md
│   ├── plan-critic/
│   │   ├── SKILL.md
│   │   └── references/
│   │       └── scoring-rubric.md
│   ├── prereq-chain/
│   │   ├── SKILL.md
│   │   └── references/
│   │       └── completion-criteria-patterns.md
│   └── confidence-gate/
│       ├── SKILL.md
│       └── references/
│           └── uncertainty-patterns.md
│
├── agents/                                  # 11 agents — 6 existing + 5 new writer agents
│   ├── dev-orchestrator.md
│   ├── architect.md
│   ├── requirements-analyst.md
│   ├── release-manager.md
│   ├── code-reviewer.md
│   ├── plan-critic-evaluator.md
│   ├── charter-writer.md                    # new (Phase 4)
│   ├── requirements-analyst.md              # already handles requirements-spec delegation
│   ├── discipline-writer.md                 # new (Phase 4)
│   ├── qa-writer.md                         # new (Phase 4)
│   ├── release-writer.md                    # new (Phase 4)
│   └── register-writer.md                   # new (Phase 4)
│
└── tests/                                   # merged test scenarios from both source plugins
    ├── arch-design/scenarios.yaml
    ├── confidence-gate/scenarios.yaml
    ├── context-keeper/scenarios.yaml
    ├── dev-discipline/scenarios.yaml
    ├── failure-modes/scenarios.yaml
    ├── plan-critic/scenarios.yaml
    ├── prereq-chain/scenarios.yaml
    ├── project-charter/scenarios.yaml
    ├── qa-strategy/scenarios.yaml
    ├── release-protocol/scenarios.yaml
    ├── requirements-spec/scenarios.yaml
    └── scope-anchor/scenarios.yaml
```

---

### Task 7.1 — Write `.claude-plugin/plugin.json`

**File:** `planitil/.claude-plugin/plugin.json`

Canonical format per Claude Code marketplace standard (name, description, version, author with name + email):

```json
{
  "name": "planitil",
  "version": "1.0.0",
  "description": "Full-stack planning discipline for Claude Code. Produces the canonical artifact chain (PROJECT.md → SCOPE.md → SPEC.md → ARCH.md → DEV-SPEC.md → QA.md → RELEASE.md) with integrated quality gates, failure-mode analysis, plan evaluation, and persistent session memory.",
  "author": {
    "name": "james"
  }
}
```

**Best practice notes:**
- `name` matches the directory name exactly — required for plugin resolution
- `version` uses semver — required for marketplace compatibility
- No `requires` field — `planitil` is fully self-contained; the dependency on `workflow-quality` is dissolved by merger

---

### Task 7.2 — Write `LICENSE`

**File:** `planitil/LICENSE`
**Content:** Apache License 2.0 (consistent with all official Claude Code marketplace plugins)

---

### Task 7.3 — Write `README.md`

**File:** `planitil/README.md`

Required sections per Claude Code README best practices (derived from marketplace plugin patterns):

```markdown
# planitil

Full-stack planning discipline for Claude Code. Combines information architecture,
quality gating, and session memory into a single coherent workflow.

## Overview

[2–3 paragraph description of the artifact chain, three-layer model, and what problem it solves]

## Skills

### Layer 1 — Information Architecture
| Skill | Produces | Prerequisite |
|---|---|---|
| `project-charter` | `PROJECT.md` | None |
| `scope-anchor` | `SCOPE.md` | PROJECT.md |
| `requirements-spec` | `SPEC.md` | PROJECT.md |
| `arch-design` | `ARCH.md`, `ADR-NNN.md` | SPEC.md |
| `dev-discipline` | `DEV-SPEC.md` | ARCH.md |
| `qa-strategy` | `QA.md` | SPEC.md + ARCH.md |
| `release-protocol` | `RELEASE.md` | QA.md |

### Layer 2 — Quality Gates
| Skill | Purpose | When to invoke |
|---|---|---|
| `scope-anchor` | Pin scope before planning | Before any plan or arch work |
| `confidence-gate` | Surface uncertainty before action | Before any significant output |
| `failure-modes` | Pre-execution failure checklist | Before planning, arch, or implementation |
| `plan-critic` | Score and revise plans | After any plan is written |
| `prereq-chain` | Gate phase transitions | During multi-phase execution |

### Layer 3 — Memory
| Skill | Produces | Invoked |
|---|---|---|
| `context-keeper` | `DECISIONS.md`, `ASSUMPTIONS.md`, `RISKS.md`, `DEBT.md` | After every significant decision |

## Agents

| Agent | Role | Tools |
|---|---|---|
| `dev-orchestrator` | Routes tasks and enforces layer sequencing | Read, Agent, Write |
| `architect` | Designs ARCH.md and ADR files | Read, Write, WebFetch |
| `requirements-analyst` | Produces SPEC.md | Read, Write, WebSearch |
| `release-manager` | Executes RELEASE.md runbook | Read, Write |
| `code-reviewer` | Reviews code against DEV-SPEC.md | Read, Grep, Glob, Write |
| `plan-critic-evaluator` | Scores plans against rubric | Read, Write |
| `charter-writer` | Writes PROJECT.md to disk | Read, Write |
| `discipline-writer` | Writes DEV-SPEC.md to disk | Read, Write |
| `qa-writer` | Writes QA.md to disk | Read, Write |
| `release-writer` | Writes RELEASE.md to disk | Read, Write |
| `register-writer` | Writes context register files to disk | Read, Write |

## Installation

[Installation steps for Claude Code plugin manager]

## Artifact Chain

```
PROJECT.md → SCOPE.md → SPEC.md → ARCH.md → DEV-SPEC.md → QA.md → RELEASE.md
                ↓             ↓          ↓
           DECISIONS.md  RISKS.md   ASSUMPTIONS.md   DEBT.md
```

## Usage

[Examples for starting a new project, running quality gates, resuming after session reset]

## Best Practices

[Link to key workflow patterns]
```

---

### Task 7.4 — Write `plugin.yaml`

**File:** `planitil/plugin.yaml`

Extended metadata format (consistent with source plugins; not required by marketplace but present in both source plugins):

```yaml
name: planitil
version: 1.0.0
description: >
  Full-stack planning discipline for Claude Code. Produces the canonical
  artifact chain: PROJECT.md → SCOPE.md → SPEC.md → ARCH.md → DEV-SPEC.md →
  QA.md → RELEASE.md. Integrates three-layer workflow: information architecture,
  quality gates, and session memory.
author: james
tags:
  - planning
  - architecture
  - requirements
  - development
  - workflow
  - quality
  - meta
skills:
  - project-charter
  - requirements-spec
  - arch-design
  - dev-discipline
  - qa-strategy
  - release-protocol
  - context-keeper
  - scope-anchor
  - confidence-gate
  - failure-modes
  - plan-critic
  - prereq-chain
agents:
  - dev-orchestrator
  - architect
  - requirements-analyst
  - release-manager
  - code-reviewer
  - plan-critic-evaluator
  - charter-writer
  - discipline-writer
  - qa-writer
  - release-writer
  - register-writer
```

**Note:** No `requires:` field — this is a self-contained package.

---

### Task 7.5 — Copy and patch skills

For each of the 12 skills, copy from the source plugin and apply any patches from earlier phases:

| Skill | Source | Patches to apply |
|---|---|---|
| `project-charter` | `dev-system` | Task 4.2 (spawn charter-writer), Task 5.2 (path to `.planning/`) |
| `requirements-spec` | `dev-system` | Task 4.3 (spawn requirements-analyst), Task 5.2 |
| `arch-design` | `dev-system` | Task 5.2 (paths), Task 6.1 (relative reference links) |
| `dev-discipline` | `dev-system` | Task 4.4 (spawn discipline-writer), Task 5.2 |
| `qa-strategy` | `dev-system` | Task 4.5 (spawn qa-writer), Task 5.2 |
| `release-protocol` | `dev-system` | Task 4.6 (spawn release-writer), Task 5.2 |
| `context-keeper` | `dev-system` | Task 4.7 (spawn register-writer), Task 5.2 |
| `scope-anchor` | `workflow-quality` | None — copy as-is |
| `failure-modes` | `workflow-quality` | None — copy as-is |
| `plan-critic` | `workflow-quality` | Task 6.3 (threshold 7.5 → 8) |
| `prereq-chain` | `workflow-quality` | None — copy as-is |
| `confidence-gate` | `workflow-quality` | None — copy as-is |

**Acceptance:** Each skill file passes the compliance checklist in Task 7.9.

---

### Task 7.6 — Copy and patch agents

For each of the 6 existing agents, copy from `~/.claude/agents/` and apply patches:

| Agent | Patches to apply |
|---|---|
| `dev-orchestrator.md` | Task 1 (no settings change needed here), Task 3.1 (routing guard), Task 5.1 (ensure .planning/ exists; add Write to tools) |
| `architect.md` | Task 1.2 (network constraint instructions) |
| `requirements-analyst.md` | Task 1.3 (network constraint instructions) |
| `release-manager.md` | Task 6.2 (conflict guard) |
| `code-reviewer.md` | Task 2.1 (add Write tool; add persistence instructions) |
| `plan-critic-evaluator.md` | Task 2.2 (add Write tool; add persistence instructions), Task 6.3 (threshold 7.5 → 8) |

**Acceptance:** Each agent file passes the compliance checklist in Task 7.9.

---

### Task 7.7 — Create the 5 new writer agents

Create each writer agent at `planitil/agents/` with the following specification. All writer agents share the same structural pattern: Read (check for existing file and read prerequisites), Write (produce the artifact).

**Common frontmatter pattern for all writer agents:**
```markdown
---
name: [name]-writer
description: |
  Receives completed [ARTIFACT] content from the [skill-name] skill and writes
  it to .planning/[ARTIFACT] at the project root. Part of the planitil
  artifact chain. Spawned by [skill-name]; do not invoke directly.
model: sonnet
tools:
  - Read
  - Write
color: [color]
---
```

| Agent | Artifact written | Color | Spawned by |
|---|---|---|---|
| `charter-writer.md` | `.planning/PROJECT.md` | cyan | `project-charter` skill |
| `discipline-writer.md` | `.planning/DEV-SPEC.md` | blue | `dev-discipline` skill |
| `qa-writer.md` | `.planning/QA.md` | green | `qa-strategy` skill |
| `release-writer.md` | `.planning/RELEASE.md` | yellow | `release-protocol` skill |
| `register-writer.md` | `.planning/DECISIONS.md`, `.planning/ASSUMPTIONS.md`, `.planning/RISKS.md`, `.planning/DEBT.md` | magenta | `context-keeper` skill |

Each writer agent must:
1. Read any prerequisite files required for validation (e.g., `charter-writer` reads existing PROJECT.md to check if an update or fresh write is needed)
2. Write the artifact to the canonical `.planning/` path
3. Confirm with a structured message: `[ARTIFACT] written to .planning/[ARTIFACT].`
4. Return `WRITER_ERROR: [reason]` if the prerequisite chain check fails

---

### Task 7.8 — Copy test scenarios

Copy `tests/` directories from both source plugins into `planitil/tests/`. No modifications needed — test scenarios are standalone YAML files that do not reference source plugin paths.

Source → Destination:
- `dev-system/tests/*` → `planitil/tests/`
- `workflow-quality/tests/*` → `planitil/tests/`

No conflicts exist; the two plugins have non-overlapping test directories.

---

### Task 7.9 — Compliance checklist

Before marking Phase 7 complete, verify every file in `planitil/` against the following checklist. This checklist is derived from Claude Code plugin best practice guidelines observed in the official marketplace.

#### `.claude-plugin/plugin.json`
- [ ] `name` field present and matches directory name
- [ ] `description` is a single coherent sentence (no line breaks)
- [ ] `version` is semver (`MAJOR.MINOR.PATCH`)
- [ ] `author.name` present
- [ ] No `requires` field (self-contained)

#### `plugin.yaml`
- [ ] `name` matches `plugin.json` name
- [ ] `skills` list names match actual directory names under `skills/`
- [ ] `agents` list names match actual file names under `agents/` (without `.md`)
- [ ] No `requires` field

#### `README.md`
- [ ] Overview section present
- [ ] Skills table covers all 12 skills
- [ ] Agents table covers all 11 agents
- [ ] Installation section present
- [ ] At least one usage example

#### `LICENSE`
- [ ] Apache 2.0 text present

#### Each `skills/*/SKILL.md`
- [ ] Frontmatter includes `name`, `description`, `version`
- [ ] `description` includes at least one `<example>` block
- [ ] "What This Skill Produces" section present
- [ ] All `references/` links are relative (no absolute paths, no `/Users/...`)
- [ ] Spawns a writer agent for any artifact it produces (or is a Layer 2 / Layer 3 quality skill)
- [ ] No reference to `workflow-quality` or `dev-system` plugin names — self-contained

#### Each `agents/*.md`
- [ ] Frontmatter includes `name`, `description`, `model`, `tools`, `color`
- [ ] `model` is one of: `inherit`, `sonnet`, `opus`, `haiku`
- [ ] `tools` list is explicit (no implicit tool inheritance)
- [ ] `color` is one of the Claude Code named colors
- [ ] Instruction body includes a "Mandatory First Action" or equivalent guard
- [ ] Error return format is a namespaced constant (e.g., `ARCHITECT_ERROR:`, `WRITER_ERROR:`)
- [ ] No hardcoded absolute paths
- [ ] No reference to `workflow-quality` or `dev-system` plugin names

#### `tests/*/scenarios.yaml`
- [ ] Each scenario file is valid YAML
- [ ] Scenario names are unique within the file

---

### Task 7.10 — Verify installability

After assembly, verify the package can be recognized by Claude Code:

**Structural check:**
```bash
# From /Users/james/src/test/
ls planitil/.claude-plugin/plugin.json    # must exist
ls planitil/skills/                       # must contain 12 directories
ls planitil/agents/                       # must contain 11 .md files
ls planitil/tests/                        # must contain 12 scenario directories
```

**Self-containment check:** Grep the entire `planitil/` tree for any reference to `workflow-quality`, `dev-system`, or `/Users/james/` — none should appear in any file that will be read at runtime (SKILL.md, agent .md files, plugin.json, plugin.yaml).

```bash
grep -r "workflow-quality\|dev-system\|/Users/james" planitil/skills/ planitil/agents/ planitil/.claude-plugin/ planitil/plugin.yaml
# Expected: no output
```

**Acceptance:** Zero matches on the self-containment grep. All structural checks pass.

---

## Updated Phasing Overview (with Phase 7)

| Phase | Focus | Findings addressed | Risk | Depends on |
|---|---|---|---|---|
| 1 | Security & Permissions | #1, #6 | Low | — |
| 2 | Persistence gaps | #2, #3 | Low | — |
| 3 | Layer 2 routing guard | #4 | Low | — |
| 4 | Skill write delegation | #7 | Medium | — |
| 5 | Path standardization | #8 | Medium | Phase 4 |
| 6 | Polish | #5, #9, #10 | Low | — |
| 7 | Assemble `planitil` package | All findings | Low | Phases 1–6 |

## Files to Create in `planitil/` (net new, Phase 7)

| File | Source | Notes |
|---|---|---|
| `planitil/.claude-plugin/plugin.json` | New | Task 7.1 |
| `planitil/LICENSE` | New | Task 7.2 — Apache 2.0 |
| `planitil/README.md` | New | Task 7.3 |
| `planitil/plugin.yaml` | New | Task 7.4 |
| `planitil/skills/project-charter/**` | dev-system | Task 7.5 + Phase 4/5 patches |
| `planitil/skills/requirements-spec/**` | dev-system | Task 7.5 + Phase 4/5 patches |
| `planitil/skills/arch-design/**` | dev-system | Task 7.5 + Phase 5/6 patches |
| `planitil/skills/dev-discipline/**` | dev-system | Task 7.5 + Phase 4/5 patches |
| `planitil/skills/qa-strategy/**` | dev-system | Task 7.5 + Phase 4/5 patches |
| `planitil/skills/release-protocol/**` | dev-system | Task 7.5 + Phase 4/5 patches |
| `planitil/skills/context-keeper/**` | dev-system | Task 7.5 + Phase 4/5 patches |
| `planitil/skills/scope-anchor/**` | workflow-quality | Task 7.5 — copy as-is |
| `planitil/skills/failure-modes/**` | workflow-quality | Task 7.5 — copy as-is |
| `planitil/skills/plan-critic/**` | workflow-quality | Task 7.5 + Phase 6 threshold patch |
| `planitil/skills/prereq-chain/**` | workflow-quality | Task 7.5 — copy as-is |
| `planitil/skills/confidence-gate/**` | workflow-quality | Task 7.5 — copy as-is |
| `planitil/agents/dev-orchestrator.md` | ~/.claude/agents | Task 7.6 + Phase 3/5 patches |
| `planitil/agents/architect.md` | ~/.claude/agents | Task 7.6 + Phase 1 patch |
| `planitil/agents/requirements-analyst.md` | ~/.claude/agents | Task 7.6 + Phase 1 patch |
| `planitil/agents/release-manager.md` | ~/.claude/agents | Task 7.6 + Phase 6 patch |
| `planitil/agents/code-reviewer.md` | ~/.claude/agents | Task 7.6 + Phase 2 patch |
| `planitil/agents/plan-critic-evaluator.md` | ~/.claude/agents | Task 7.6 + Phase 2/6 patches |
| `planitil/agents/charter-writer.md` | New | Task 7.7 |
| `planitil/agents/discipline-writer.md` | New | Task 7.7 |
| `planitil/agents/qa-writer.md` | New | Task 7.7 |
| `planitil/agents/release-writer.md` | New | Task 7.7 |
| `planitil/agents/register-writer.md` | New | Task 7.7 |
| `planitil/tests/**` | dev-system + workflow-quality | Task 7.8 — copy as-is |
