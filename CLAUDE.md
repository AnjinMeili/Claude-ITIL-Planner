# CLAUDE.md — planitil

planitil is a Claude Code plugin. There is no runtime, no build step, no package manager. The entire system is Markdown and YAML consumed by the Claude Code plugin loader.

---

## CRITICAL: settings.local.json contains stale paths

`.claude/settings.local.json` contains hardcoded `Bash(cp ...)` permissions pointing to `/Users/james/...`. These are from the original dev machine and will not work on any other machine. Do not run these commands. Do not commit this file as-is. Before using it on a new machine, replace all `/Users/james/` path prefixes with the correct local path, or clear the `allow` array entirely and re-add only what is needed.

---

## Repository structure

```
planitil/
  plugin.yaml                  # Root plugin manifest (name, version, skills dir, agents list)
  .claude-plugin/plugin.json   # Alternate manifest format (kept in sync with plugin.yaml)
  skills/{name}/
    SKILL.md                   # Skill definition — frontmatter + body
    references/                # Rubrics, templates, sub-folders consumed by the skill
      assets/                  # Templates (*.md.template)
  agents/{name}.md             # Agent definitions — frontmatter + body
  tests/{name}/
    scenarios.yaml             # Test scenarios for the named skill
  TODO.md                      # Known issues from production observations
```

All artifacts produced at runtime land in `.planning/` at the consuming project root, not in this repo.

---

## Skill authoring rules

Every `SKILL.md` requires YAML frontmatter with these fields:

```yaml
---
name: skill-name          # kebab-case, matches directory name exactly
description: |            # Multi-line. Must include at least one <example> block.
  ...
  <example>
  User: "..."
  Action: ...
  </example>
version: 1.0.0
---
```

Optional frontmatter fields:
- `argument-hint: [description]` — required when the skill accepts a direct argument (e.g., `plan-critic` takes a plan file path). Omit if the skill gathers context itself.

Rules:
- `name` must match the directory name. Discovery is directory-based; a mismatch breaks skill lookup.
- `description` drives invocation matching. Weak descriptions = the skill never fires. Every description needs at least one `<example>` block showing a real trigger.
- Do not add frontmatter fields not listed above without verifying plugin loader support.

---

## Agent authoring rules

Every agent `.md` requires YAML frontmatter:

```yaml
---
name: agent-name
description: |
  ...
  <example>...</example>
model: inherit | haiku | sonnet | opus
tools:
  - Read
  - Write
  - Agent
  - Bash
  - WebFetch
color: magenta | blue | yellow | green | red | cyan
---
```

Model selection guidance:
- `inherit` — orchestrators only (dev-orchestrator). Inherits the calling context's model.
- `haiku` — evaluators running scoring loops (plan-critic-evaluator). Fast and cheap; acceptable for rubric-based scoring against well-defined criteria. Known limitation: misses subtle cross-artifact violations (see TODO.md — consider Sonnet for second revision cycle).
- `sonnet` — all substantive agents (architect, requirements-analyst, code-reviewer, release-manager). Default choice.
- `opus` — not currently used. Justify explicitly before adding.

Tool access — grant minimum necessary:
- `Read` — universal
- `Write` — only agents that produce artifacts
- `Agent` — only orchestrators that spawn subagents
- `Bash` — only agents that need shell access (currently none)
- `WebFetch` — architect only (API spec lookups); scope is tightly restricted in the agent body

---

## Artifact chain and ownership

Layer 1 (information architecture) — produced in sequence, each gates the next:

```
PROJECT.md → SCOPE.md → SPEC.md → ARCH.md + ADR-NNN.md → DEV-SPEC.md
```

Layer 2 (quality gates) — run after Layer 1 artifacts exist:

```
failure-modes → plan-critic → prereq-chain (per phase) → confidence-gate (per decision)
```

Layer 3 (memory) — run after every significant decision:

```
context-keeper → DECISIONS.md, ASSUMPTIONS.md, RISKS.md, DEBT.md
```

All runtime artifacts write to `.planning/` at the consuming project root. This repo owns no `.planning/` directory and no runtime state.

**Artifact written vs. artifact confirmed are different states.** A file on disk is not confirmed. Confirmed means all open items (OI-xx) resolved or explicitly deferred, and the initiating party has acknowledged. Downstream skills treat file presence as confirmation — this is a known gap (see TODO.md, requirements-spec section). Do not contribute code that widens this gap.

---

## Sequencing rules (enforced by dev-orchestrator, replicated here for reference)

| Requested action | Hard prerequisites |
|---|---|
| `requirements-spec` | PROJECT.md exists |
| `arch-design` | SPEC.md confirmed |
| `dev-discipline` | ARCH.md exists |
| `failure-modes` or `plan-critic` | SCOPE.md exists + ARCH.md or PLAN.md exists |
| `qa-strategy` | SPEC.md + ARCH.md + DEV-SPEC.md all exist |
| `release-protocol` | QA.md exists |

Scope-anchor must run before plan-critic. If plan-critic is requested and SCOPE.md is absent, redirect to scope-anchor first — do not proceed and apply the penalty silently.

All Layer 1 artifacts including SCOPE.md are written to `.planning/`. This is the canonical location the orchestrator and plan-critic check. Writing SCOPE.md to the project root instead of `.planning/` will silently break prerequisite detection.

---

## Naming conventions

Used across all runtime artifacts. Do not invent new prefixes.

| Prefix | Used in | Format |
|---|---|---|
| `UC-NN` | Use cases | UC-01, UC-02 |
| `AC-NN-MM` | Acceptance criteria | AC-01-01 (UC-01, criterion 1) |
| `DEC-NNN` | DECISIONS.md | DEC-001 |
| `ASS-NNN` | ASSUMPTIONS.md | ASS-001 |
| `RISK-NNN` | RISKS.md | RISK-001 |
| `DEBT-NNN` | DEBT.md | DEBT-001 |
| `ADR-NNN` | Architecture decisions | ADR-001.md |
| `OI-NN` | Open items in SPEC.md | OI-01 |

---

## ADR status discipline

New ADRs must default to `Status: Proposed`. Transition to `Accepted` only after confidence-gate passes or explicit user confirmation. The current codebase has examples of `Status: Accepted` set immediately on creation — this is a known bug (see TODO.md, arch-design section). Do not propagate this pattern.

---

## Test scenario authoring

Scenarios live at `planitil/tests/{skill-name}/scenarios.yaml`. Required fields:

```yaml
kind: skill
ref: skills/{skill-name}       # Must match the skills/ directory name exactly
scenarios:
  - name: kebab-case-name
    description: one sentence
    prompt: |
      The trigger input for this scenario.
    expectations:
      keywords:
        - "string that must appear in output"
    judge:
      criteria: |
        Score on:
        - Criterion A (Npts)
        - Criterion B (Npts)
      passing_score: 7.0       # Float. Typically 7.0 out of 10.
    timeout: 60                # Seconds. Use 60 for most skills; increase for multi-step flows.
    runs: 2                    # Minimum runs before result is stable.
```

- `ref` is a path relative to the `planitil/` directory — it must match an actual skill directory.
- `passing_score` is a float. Point allocations in `criteria` must sum to a value where `passing_score` is achievable.
- `keywords` are exact substring matches. Use output tokens the skill is specified to produce (e.g., `"ARCH.md"`, `"Alternatives Rejected"`), not natural language paraphrases.

---

## What not to do

- **Do not simulate a missing skill inline.** If a skill is unavailable, the dev-orchestrator returns a `ROUTING_ERROR` and stops. Do not attempt to replicate skill behavior in a general response.
- **Do not write downstream artifacts before upstream ones are confirmed.** Writing ARCH.md before SPEC.md is confirmed creates compounding errors.
- **Do not add `.planning/` to this repo.** It is a runtime output directory for consuming projects, not plugin source.
- **Do not add build artifacts, lock files, or language-specific config.** This is a pure content repo. No `package.json`, `pyproject.toml`, `Makefile`, or equivalent belongs here.
- **Do not keep both manifests out of sync.** `plugin.yaml` and `.claude-plugin/plugin.json` must list identical agents and skills. When adding or removing either, update both files.

---

## Known issues (from TODO.md — relevant to contributors)

These are production-observed gaps, not hypotheticals. Do not make them worse.

- **plan-critic-evaluator receives no path to the failure-modes catalog.** Anti-Pattern scoring runs from the evaluator's general knowledge, not from `references/failure-modes/planning.md`. Fix: pass the catalog path explicitly in the evaluator prompt.
- **ADRs written with `Status: Accepted` immediately.** Should default to `Proposed`. See arch-design SKILL.md — the text is correct but unenforced.
- **context-keeper not auto-invoked after decisions.** The dev-orchestrator specifies this as post-processing but it does not happen automatically. All session decisions go unrecorded unless explicitly triggered.
- **dev-discipline produces no `.pre-commit-config.yaml`.** DEV-SPEC.md declares pre-commit hooks; the skill never creates the config file.
- **DEBT.md stub not created by dev-discipline.** First debt item has no file to append to.
- **scope-anchor must precede plan-critic** — the orchestrator routing table says this but does not enforce it as a hard gate.

---

## Git hygiene

This is a pure content repo. Commit messages should describe what changed in the plugin behavior, not file mechanics.

Good: `fix: plan-critic scoring rubric — clarify Scope Fit auto-penalty output format`
Weak: `update SKILL.md`

Do not commit:
- `.planning/` directories or any runtime artifacts
- `.claude/settings.local.json` with stale `/Users/james/` paths (see top of this file)
- Editor configs, OS metadata (`.DS_Store`), or IDE project files
