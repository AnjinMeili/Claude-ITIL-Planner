# Plan Review: dev-system-remediation-plan.md

**Date:** 2026-04-01
**Plan reviewed:** `dev-system-remediation-plan.md`
**Reviewer:** Claude Code (plan-critic rubric)

---

## Plan Critic Scorecard

| Dimension | Score | Rationale |
|---|---|---|
| **Goal Alignment** | 2/2 | All 10 findings from the review map to a task. Every phase header names which finding(s) it addresses. Phase 7 traces directly to the user-stated packaging goal. No orphaned tasks found. |
| **Completeness** | 1/2 | Task 4.3 is explicitly unresolved ("may be redundant — audit first"), leaving an open branch with no acceptance criterion. Task 7.7 compresses "create 5 writer agents" into a frontmatter template + table but does not specify instruction body content for any agent — the primary intellectual work is hand-waved. Task 7.3 README has three literal placeholders (`[2–3 paragraph description...]`, `[Examples...]`, `[Link to...]`). The directory tree has a duplicate `requirements-analyst.md` entry. DEV-SPEC.md location exception is documented but never resolved. |
| **Risk Surface** | 1/2 | Task 1.1 changes a global setting that affects all plugins system-wide, not just dev-system — no pre-check or rollback defined. The content-transfer mechanism for Phase 4 (how a skill passes drafted content to its writer agent) is unspecified — this is an architectural gap that could make all 5 writer agents non-functional. No transition plan for users who already have `dev-system` and `workflow-quality` installed when they install `planitil`. |
| **Scope Fit** | 2/2 | All tasks trace to findings or the packaging goal. The addition of `Write` to `dev-orchestrator`'s tools list is a side effect of Phase 5 and is noted. No out-of-scope work detected. |
| **Anti-Pattern** | 1/2 | **Single-Step Compression** in Task 7.7 — "create 5 writer agents" hides the full instruction-body authoring for each agent. Task 7.9 compliance checklist flags "What This Skill Produces section present" against all 12 skills, but 5 of the Layer 2 skills (scope-anchor, failure-modes, etc.) don't have this section by design — the carve-out is present but buried. **Implicit Dependency** — Phase 7 assembly depends on Phases 1–6 being applied to the *source files*, but no gate verifies this before Phase 7 begins. |
| **Total** | **7/10** | |

**Verdict: FAIL** — below threshold of 8.

---

## Revision Required — Completeness

### R1 — Task 4.3 open branch

Resolve the ambiguity before execution. Check whether `requirements-spec` SKILL.md already contains a spawn instruction for `requirements-analyst`.

- If yes: Task 4.3 becomes "Verify the spawn instruction exists; no new agent file needed."
- If no: Task 4.3 becomes "Add spawn instruction to `requirements-spec` SKILL.md pointing at `requirements-analyst`."

Either way, close the branch with a defined outcome and acceptance criterion. The task cannot remain as a conditional with no stated resolution.

---

### R2 — Task 7.7 Single-Step Compression

"Create 5 writer agents" compresses full instruction-body authoring into a frontmatter template and a table. Expand each of the 5 writer agents beyond the template. For each agent, specify:

- What prerequisite files it reads before writing (e.g., `charter-writer` checks whether `.planning/PROJECT.md` already exists to determine update vs. create)
- What validation it performs on the incoming content before writing (e.g., refuse to write an empty or stub file)
- The exact confirmation message format
- The `WRITER_ERROR:` conditions that cause it to refuse to write

---

### R3 — Task 7.3 README placeholders

Three sections contain literal placeholder text that cannot be executed:
- `[2–3 paragraph description of the artifact chain, three-layer model, and what problem it solves]`
- `[Examples for starting a new project, running quality gates, resuming after session reset]`
- `[Link to key workflow patterns]`

Replace each with explicit content instructions stating what the section must contain. Placeholder text at assembly time will produce an incomplete README.

---

### R4 — Duplicate `requirements-analyst.md` in directory tree

The `agents/` tree in Phase 7 lists `requirements-analyst.md` twice (once at the correct position, once after `charter-writer.md`). Remove the duplicate. If `spec-writer.md` was intended at that position, name it correctly and reconcile with the Task 4.3 resolution. The agents tree must match the agents table in Task 7.7 exactly — 11 unique entries.

---

### R5 — DEV-SPEC.md location exception unresolved

Phase 5 canonical path table lists `.planning/DEV-SPEC.md` then immediately notes "Accept either `.planning/DEV-SPEC.md` or project root as valid." This exception is not resolved — it defers a decision that both Task 4.4 (`discipline-writer` output path) and Task 5.2 (path updates) depend on.

**Recommended resolution:** Canonicalize to `.planning/DEV-SPEC.md`. Add a note in `dev-discipline` SKILL.md that `code-reviewer` checks both `.planning/DEV-SPEC.md` and project root for backward compatibility. Remove the "either is valid" ambiguity from the canonical path table.

---

## Revision Required — Risk Surface

### R6 — Task 1.1: no pre-check or rollback for global settings change

`skipDangerousModePermissionPrompt: true` is a global setting applied to all plugins and hooks in `~/.claude/`, not scoped to dev-system. Removing it may break other workflows.

Add before Task 1.1:
- **Pre-check:** Audit `~/.claude/hooks/`, `~/.claude/plugins/`, and any active session configurations for workflows that currently depend on the permission bypass.
- **Rollback:** If removing the flag breaks other workflows, restore it and address the permission issue per-agent using an `allowedTools` scoping mechanism rather than a global bypass.

---

### R7 — Writer agent content-transfer mechanism unspecified

Phase 4 creates writer agents but never specifies how a skill — running inside `dev-orchestrator`'s tool context — passes the drafted content to the writer agent at spawn time. Without this, Phase 4 is architecturally unimplementable.

The three candidate mechanisms are:

| Option | Mechanism | Implication |
|---|---|---|
| A | Skill writes draft to `.planning/draft-[artifact].md`; writer agent reads, validates, and finalizes | Requires two writes; leaves draft files on disk |
| B | Skill passes content inline in the agent spawn prompt | Content size limited by prompt; no intermediate file |
| C | `dev-orchestrator` holds `Write`; skill+orchestrator combination writes directly without a separate writer agent | Writer agents become redundant; simplifies architecture |

**One option must be chosen and stated explicitly.** Option C is worth re-evaluating: if `dev-orchestrator` already has `Write` (required by Task 5.1), adding writer agents as intermediaries may be unnecessary architectural complexity.

---

### R8 — No transition plan for existing installations

Phase 7 assembles `planitil` but does not address users who already have `dev-system` and `workflow-quality` installed. Both plugins register the same skill names that `planitil` registers. Without a transition plan, installing `planitil` creates duplicate skill registrations.

Add to Phase 7 prerequisites:
- State whether `dev-system` and `workflow-quality` should be uninstalled before installing `planitil`
- Confirm whether the Claude Code plugin manager handles skill name collisions or silently routes to the first match
- Specify how to verify no duplicate skill invocations occur post-install

---

## Revision Required — Anti-Pattern

### R9 — Task 7.9 missing cross-reference check

The compliance checklist in Task 7.9 does not verify that agent names referenced inside skill SKILL.md files resolve to actual agent files in `planitil/agents/`. For example, `plan-critic` SKILL.md references `plan-critic-evaluator` by name. If that agent was excluded from the package or renamed, the skill would fail silently at runtime.

Add to the Task 7.9 compliance checklist under **Each `skills/*/SKILL.md`**:
```
- [ ] All agent names referenced by name in skill body resolve to an actual
      .md file in planitil/agents/ (grep for "agent" references and cross-check)
```

---

### R10 — Phase 7 gate on Phase 1–6 completion is implicit

The plan states "Phase 7 depends on Phases 1–6 being complete" but provides no gate check to verify this before Phase 7 begins. If an executor starts Phase 7 with unpatched source files, the assembly silently produces the pre-remediation versions of all artifacts.

Add an explicit pre-flight check at the start of Task 7.5:

```
Before copying any source files, verify all Phase 1–6 patches are applied:
- [ ] ~/.claude/settings.json: skipDangerousModePermissionPrompt is absent or false
- [ ] ~/.claude/agents/code-reviewer.md: tools list includes Write
- [ ] ~/.claude/agents/plan-critic-evaluator.md: tools list includes Write
- [ ] ~/.claude/agents/charter-writer.md: file exists
- [ ] ~/.claude/agents/discipline-writer.md: file exists
- [ ] ~/.claude/agents/qa-writer.md: file exists
- [ ] ~/.claude/agents/release-writer.md: file exists
- [ ] ~/.claude/agents/register-writer.md: file exists
- [ ] ~/.claude/plugins/dev-system/skills/arch-design/SKILL.md: no /Users/ paths
- [ ] ~/.claude/plugins/workflow-quality/skills/plan-critic/SKILL.md: threshold is 8 not 7.5

If any check fails, stop and complete the relevant Phase 1–6 task before continuing.
```

---

## Summary of Required Changes to Plan

| Revision | Affects | Type |
|---|---|---|
| R1 — Resolve Task 4.3 open branch | Phase 4, Task 4.3 | Completeness |
| R2 — Expand Task 7.7 writer agent bodies | Phase 7, Task 7.7 | Completeness / Single-Step Compression |
| R3 — Replace Task 7.3 README placeholders | Phase 7, Task 7.3 | Completeness |
| R4 — Fix duplicate entry in directory tree | Phase 7, Target Structure | Completeness |
| R5 — Resolve DEV-SPEC.md location ambiguity | Phase 4, Phase 5 | Completeness |
| R6 — Add pre-check + rollback for Task 1.1 | Phase 1, Task 1.1 | Risk Surface |
| R7 — Specify writer agent content-transfer mechanism | Phase 4 | Risk Surface |
| R8 — Add installation transition plan | Phase 7 | Risk Surface |
| R9 — Add agent cross-reference check to Task 7.9 | Phase 7, Task 7.9 | Anti-Pattern |
| R10 — Add Phase 7 pre-flight gate | Phase 7, Task 7.5 | Anti-Pattern / Implicit Dependency |

**Minimum to reach PASS (score ≥ 8):** R1, R2, R5 (Completeness to 2/2) + R7 (Risk Surface to 2/2) would bring the score to 9/10 with Anti-Pattern remaining at 1/2. R2 also partially addresses the Anti-Pattern dimension. All 10 revisions are recommended before execution begins.
