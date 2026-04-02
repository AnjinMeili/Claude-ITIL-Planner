# Code Review: dev-system plugin + downstream agents

**Date:** 2026-04-01
**Scope:** `~/.claude/plugins/dev-system/` (plugin.yaml, 7 skills), `~/.claude/agents/` (5 downstream agents), `~/.claude/settings.json`
**Reviewer:** Claude Code

---

## Scope Inventory

| Component | Type | Path |
|---|---|---|
| `plugin.yaml` | Plugin manifest | `~/.claude/plugins/dev-system/plugin.yaml` |
| `project-charter` | Skill | `skills/project-charter/SKILL.md` |
| `requirements-spec` | Skill | `skills/requirements-spec/SKILL.md` |
| `arch-design` | Skill | `skills/arch-design/SKILL.md` |
| `dev-discipline` | Skill | `skills/dev-discipline/SKILL.md` |
| `qa-strategy` | Skill | `skills/qa-strategy/SKILL.md` |
| `release-protocol` | Skill | `skills/release-protocol/SKILL.md` |
| `context-keeper` | Skill | `skills/context-keeper/SKILL.md` |
| `dev-orchestrator` | Agent | `~/.claude/agents/dev-orchestrator.md` |
| `architect` | Agent | `~/.claude/agents/architect.md` |
| `requirements-analyst` | Agent | `~/.claude/agents/requirements-analyst.md` |
| `release-manager` | Agent | `~/.claude/agents/release-manager.md` |
| `code-reviewer` | Agent | `~/.claude/agents/code-reviewer.md` |
| `plan-critic-evaluator` | Agent | `~/.claude/agents/plan-critic-evaluator.md` |
| `settings.json` | Configuration | `~/.claude/settings.json` |

---

## Critical

### 1. `~/.claude/settings.json` — `skipDangerousModePermissionPrompt: true`

`"skipDangerousModePermissionPrompt": true` disables all tool-use permission prompts globally. Every agent in this pipeline — including those with `WebFetch` and `WebSearch` — runs without any user confirmation gate. Combined with agents that can write to the filesystem, this is a blanket bypass on all agent actions across every project.

---

### 2. `agents/code-reviewer.md` — No Write tool; review never persisted

The code-reviewer has `tools: Read, Grep, Glob` — no `Write`. The structured review output (`## Code Review: ...`) is returned as agent text only and is **never written to disk**. If the intent is a persisted review artifact (e.g., `REVIEW.md` or `CODE-REVIEW.md`), this agent cannot produce it.

---

### 3. `agents/plan-critic-evaluator.md` — No Write tool; scorecard never persisted

Same issue. `tools: Read` only. The `PLAN_CRITIC_SCORECARD` block is returned as agent output text but never written to a file. No `SCORECARD.md` or similar artifact is produced or persisted to the filesystem.

---

## Important

### 4. `agents/dev-orchestrator.md` — References non-existent Layer 2 skills

The orchestrator routes to `scope-anchor`, `failure-modes`, `prereq-chain`, `confidence-gate`, and `plan-critic` — but **none of these skills exist** in the plugin. The plugin manifest declares 7 skills; all are Layer 1 / Layer 3. The entire Layer 2 quality-gate layer is referenced but absent. Routing to non-existent skills will silently fail or fall through to the LLM without structured execution.

---

### 5. `skills/arch-design/SKILL.md` — Hardcoded absolute paths in References section

```
/Users/james/.claude/plugins/dev-system/skills/arch-design/references/adr-patterns.md
/Users/james/.claude/plugins/dev-system/skills/arch-design/references/system-boundary-guide.md
/Users/james/.claude/plugins/dev-system/skills/arch-design/references/assets/ADR.md.template
```

These paths are hardcoded to a specific user's home directory. All other skills use relative `references/` links. This skill is the only outlier and would produce broken references on any other machine or user account.

---

### 6. `agents/architect.md` + `agents/requirements-analyst.md` — Unrestricted external network access

- `architect.md` declares `tools: Read, Write, WebFetch`
- `requirements-analyst.md` declares `tools: Read, Write, WebSearch`

Both can reach external network resources with no scope or domain restriction. With `skipDangerousModePermissionPrompt: true` in settings.json, these execute without user confirmation. Neither agent's instructions constrain what URLs or queries are permitted.

---

### 7. Skills have no `tools:` declarations — filesystem writes depend entirely on calling context

Skills (`SKILL.md` files) carry no `tools:` frontmatter. They execute within the calling agent's tool context. This means a skill invoked from `dev-orchestrator` (which has only `tools: Read, Agent`) **cannot write files**, silently breaking the artifact production for:

| Skill | Expected artifact | Can write via dev-orchestrator? |
|---|---|---|
| `project-charter` | `PROJECT.md` | ❌ No |
| `requirements-spec` | `SPEC.md` | ❌ No |
| `arch-design` | `ARCH.md`, `ADR-NNN.md` | ❌ No |
| `dev-discipline` | `DEV-SPEC.md` | ❌ No |
| `qa-strategy` | `QA.md` | ❌ No |
| `release-protocol` | `RELEASE.md` | ❌ No |
| `context-keeper` | `DECISIONS.md`, `ASSUMPTIONS.md`, `RISKS.md`, `DEBT.md` | ❌ No |

The pattern that works is `arch-design` (skill) → spawns `architect` (agent with Write). Not all skills follow this pattern.

---

## Suggestions

### 8. Output path inconsistency across the pipeline

Artifacts are scattered between `.planning/` and project root with inconsistent fallback logic:

| Artifact | Declared location |
|---|---|
| `PROJECT.md` | Project root |
| `SPEC.md` | `.planning/` or project root |
| `ARCH.md` + ADRs | `.planning/` (architect agent) |
| `DEV-SPEC.md` | Project root |
| `QA.md` | Project root |
| `RELEASE.md` | Project root |
| `DECISIONS.md`, `ASSUMPTIONS.md`, `RISKS.md`, `DEBT.md` | `.planning/` or project root |

The "or project root" fallback makes artifact location non-deterministic without `.planning/` pre-existing. Worth standardizing and having `dev-orchestrator` ensure the directory exists before routing.

---

### 9. `release-manager` — No concurrency guard on inline RELEASE.md writes

The agent correctly has Write access and writes gate clearance timestamps back into `RELEASE.md`. However, if two release-manager instances ran against the same file simultaneously, the last write wins silently with no conflict detection.

---

### 10. `plan-critic-evaluator` — Threshold 7.5 is unreachable by integer scoring

The verdict rule is `If Total >= 7.5: PASS`. With 5 dimensions scored 0, 1, or 2, all totals are whole integers (max 10). A score of 7.5 can never be achieved — the effective PASS threshold is 8/10 (≥ 80%). Worth clarifying the intended threshold in the rubric.

---

## Positive

The artifact chain is clearly sequenced and every agent enforces its prerequisites before acting (`ARCHITECT_ERROR`, `ANALYST_ERROR`, `RELEASE_ERROR`). The fail-fast contract pattern — stop and return a structured error rather than attempting to proceed on incomplete inputs — is consistently applied across all agents and will produce clear, actionable failure messages during orchestration. The `context-keeper` cross-register linking model (DEC → RISK → ASS) is well-designed for session continuity.

---

## Filesystem Persistence Summary

| Component | Writes markdown to disk? | Notes |
|---|---|---|
| `project-charter` skill | ⚠️ Context-dependent | No tool declaration; needs Write in calling agent |
| `requirements-spec` skill | ⚠️ Context-dependent | No tool declaration |
| `requirements-analyst` agent | ✅ Yes | Write tool declared |
| `arch-design` skill | ⚠️ Context-dependent | Delegates to `architect` agent correctly |
| `architect` agent | ✅ Yes | Write + WebFetch tools declared |
| `dev-discipline` skill | ⚠️ Context-dependent | No tool declaration |
| `qa-strategy` skill | ⚠️ Context-dependent | No tool declaration |
| `release-protocol` skill | ⚠️ Context-dependent | No tool declaration |
| `release-manager` agent | ✅ Yes | Write tool; updates RELEASE.md in-place |
| `context-keeper` skill | ⚠️ Context-dependent | No tool declaration |
| `code-reviewer` agent | ❌ No | No Write tool; review output not persisted |
| `plan-critic-evaluator` agent | ❌ No | No Write tool; scorecard not persisted |
| `dev-orchestrator` agent | ✅ Intentional | Orchestrates only; delegates to agents with Write |

---

## Out-of-Bounds Access Summary

| Component | Can access outside working dir / `~/.claude/`? | Details |
|---|---|---|
| `architect` agent | ✅ Yes — external network | `WebFetch` with no URL restriction |
| `requirements-analyst` agent | ✅ Yes — external network | `WebSearch` with no query restriction |
| `arch-design` SKILL.md references | ⚠️ Hardcoded user path | Absolute paths to `/Users/james/.claude/...` |
| All agents | ⚠️ No permission prompts | `skipDangerousModePermissionPrompt: true` in settings.json |
| All other components | ✅ Contained | Read/Write scoped to project directory |
