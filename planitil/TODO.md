# planitil TODO

Items are grouped by layer. Each item includes the source observation that motivated it.

---

## Agent / Job Reliability

- [ ] **Watchdog for background shell commands.** After spawning a background shell command, poll or await completion confirmation before proceeding. Include a validation step that checks the job actually completed its task (verify expected output exists, exit code was 0, output file is non-empty) rather than relying solely on the process exit signal. Background jobs dispatched without a watchdog can appear to succeed while silently failing or hanging indefinitely.
  *Observed: 7 pytest processes were dispatched as background jobs and hung on an infinite loop bug; the orchestrating agent treated them as in-flight and kept spawning new ones. The infinite loop went undetected for several minutes.*

- [ ] **Subprocess timeout enforcement in shell tools.** When a Bash tool invocation runs a potentially blocking command (test suite, long compilation, network call), enforce a timeout and treat timeout as a failure — not a stall. The current pattern of re-submitting the same command when it doesn't return creates duplicate background processes.
  *Observed: Same pytest command submitted 7 times because each invocation silently dispatched to background without returning output.*

- [ ] **Deduplication check before spawning.** Before spawning a new background job, check whether an identical command is already running (via `ps` or output file existence). Prevents the multi-process pile-up pattern observed during the db_writer test loop.

---

## Skill: plan-critic

- [ ] **Add `failure-modes` invocation to the evaluator prompt.** The plan-critic skill says it should consult the `failure-modes` catalog for the Anti-Pattern dimension, but in practice the evaluator agent (`plan-critic-evaluator`) receives no reference to the catalog files. The scoring on Anti-Pattern presence relies on the evaluator's general knowledge rather than reading `references/failure-modes/planning.md`. Pass the catalog path explicitly in the evaluator prompt.
  *Observed: plan-critic passed a plan with no explicit failure-modes pre-scan; the Anti-Pattern dimension was scored from memory rather than from the catalog.*

- [ ] **Scope Fit auto-penalty should surface explicitly.** When the -1 auto-penalty fires (no SCOPE.md), the scorecard should state the pre-penalty score alongside the penalised score so the author knows exactly what was lost and why. Currently the total just appears lower with a watch note.
  *Observed: First plan-critic run produced 9/10; the -1 deduction for missing SCOPE.md was not immediately obvious from the scorecard.*

- [ ] **Re-score gate after SCOPE.md is created.** When scope-anchor is run after plan-critic has already passed, the plan-critic skill notes "re-scoring an already-passing plan wastes cycles." But a plan that passed 9/10 with a penalty may score 10/10 after the penalty is removed — which matters for downstream confidence. Consider a lightweight "penalty-lift check" that only re-evaluates the Scope Fit dimension rather than a full re-score.

---

## Skill: scope-anchor

- [ ] **Invoke scope-anchor before plan-critic, not after.** In this session scope-anchor was run after plan-critic had already produced a result. The skill docs say it should gate plan creation, but there is no enforcement mechanism. The `dev-orchestrator` routing table should include a check: if plan-critic is invoked and no SCOPE.md exists, redirect to scope-anchor first.
  *Observed: plan-critic → scope-anchor → plan-critic (second run) was the actual order.*

---

## Skill: requirements-spec / requirements-analyst agent

- [ ] **Surface open items (OI-xx) to the user before proceeding to arch-design.** The SPEC.md produced contained 8 open items requiring confirmation. The skill confirmed the artifact as done without prompting the user to resolve them. High-consequence open items (OI-05 Postgres topology, OI-08 privilege level) should gate the arch-design step.
  *Observed: OI-05 and OI-08 were resolved by follow-up user questions, not by a gate in the skill.*

- [ ] **Validate that open items are resolved before writing SPEC.md as "confirmed."** Distinguish between "SPEC.md written" (file on disk) and "SPEC.md confirmed" (all OI items resolved or explicitly deferred). Downstream skills (arch-design, plan-critic) treat the presence of SPEC.md as confirmation — this creates false confidence when high-consequence items are still open.

---

## Skill: arch-design / architect agent

- [ ] **Include failure-modes invocation as a mandatory step, not advisory.** The arch-design skill says to invoke failure-modes before finalising ARCH.md but frames it as an integration step. In practice the architect agent worked from the inline list provided in the prompt rather than running the failure-modes skill. Make this a hard prerequisite: arch-design should refuse to write ARCH.md until failure-modes has been run for the architecture type.
  *Observed: Failure modes were addressed well because they were embedded in the architect agent prompt — but only because the orchestrating agent pre-loaded them. The skill itself has no enforcement.*

- [ ] **ADR status should default to "Proposed" and require explicit acceptance.** All four ADRs were written with `Status: Accepted` immediately. The confidence-gate and plan-critic skills exist precisely to catch premature acceptance. ADRs should start as `Proposed` and transition to `Accepted` only after a confidence-gate pass or explicit user confirmation.

---

## Skill: dev-discipline

- [ ] **Add a pre-commit hook setup step.** DEV-SPEC.md says `pre-commit hooks run black --check, ruff, and commitlint on every commit` but the skill produces no `pre-commit` config file and no setup instructions. The hooks described in the spec never get installed. Add a `.pre-commit-config.yaml` template or a setup step to the skill output.
  *Observed: DEV-SPEC.md declared commitlint and ruff hooks; none were present in the generated project.*

- [ ] **Include a `DEBT.md` stub.** DEV-SPEC.md defines a debt management policy with a DEBT.md format, but the skill does not create the file. Any project using dev-discipline starts with no DEBT.md, meaning the first intentional debt item has no home. Create an empty `DEBT.md` with the header format as part of the skill output.

---

## Skill: plan-critic-evaluator (agent)

- [ ] **Model is Haiku — consider Sonnet for Final scoring.** The evaluator is a Haiku-class agent. This is efficient for routine scoring but may miss subtle anti-patterns or interface contract violations that require deeper reasoning. Consider using Sonnet for the second revision cycle (when the plan has already failed once) where false negatives are more expensive.
  *Observed: The FK ordering bug (write_readings before upsert_host) was caught by the Opus code reviewer but not by the plan-critic evaluator, even though it was traceable from the interface contracts in ARCH.md.*

---

## Skill: code-reviewer (agent)

- [ ] **Accept a diff or file list rather than requiring full file paths.** Currently the code-reviewer is invoked with an explicit list of every file to read. For larger projects this prompt becomes unwieldy. The agent should accept a git diff, a glob pattern, or a PR number and derive the file list itself.

- [ ] **Add a "first-run checklist" for greenfield projects.** The current review caught thread-safety, FK ordering, and exception leakage. These are all in the Critical/Important tier. A greenfield-specific checklist (checked before the general review) that includes: FK constraint ordering, concurrent request safety, exception sanitisation in error responses, and credential exposure would catch these class of issues faster and more reliably.

---

## Skill: context-keeper

- [ ] **Context-keeper was not invoked during this session.** The skill is designed to record decisions, assumptions, risks, and deferred items into living registers. None of the significant decisions made during this session (Postgres topology, SSH key exchange, sudoers privilege model, FK ordering fix) were recorded in DECISIONS.md. The dev-orchestrator should invoke context-keeper automatically after each confirmed ADR and after each resolved open item in SPEC.md.
  *Observed: The session produced 4 ADRs, 3 resolved OI items, and 1 post-review design fix — none were persisted to DECISIONS.md or DEBT.md.*

---

## Skill: qa-strategy

- [ ] **qa-strategy was not run during this session.** The skill produces QA.md covering test strategy, coverage targets, UAT criteria, and observability specification. It was skipped — implementation began directly after DEV-SPEC.md. For future sessions: qa-strategy should be invoked between DEV-SPEC.md and implementation. The 70 unit tests and 9 integration tests written ad-hoc during implementation were good, but without a QA.md there is no explicit coverage target, no UAT definition, and no observability specification to check against.

---

## Orchestration (dev-orchestrator)

- [ ] **Enforce layer sequencing as a hard gate, not advisory.** The dev-orchestrator routing table describes layer sequencing but the agent in this session allowed scope-anchor to be skipped until after plan-critic ran. The layer check should be a hard prerequisite: if an artifact from Layer 2 (quality gates) is requested and a Layer 1 artifact (SCOPE.md) is missing, the orchestrator should block and redirect — not proceed and note the penalty.

- [ ] **Add session resume detection.** When a session starts with `.planning/` already populated, the orchestrator should read the existing artifacts, determine the current phase, and offer a "here is where we are, here is the next logical step" summary before accepting a new command. Currently a fresh session invocation requires re-orienting manually.
  *Observed: The session was productive but each skill invocation required the user to know the correct sequence. A resume protocol would lower that cognitive load.*

- [ ] **Track which skills have been run in the current session.** The plan-critic skill explicitly says "do not re-score an already-passing plan — wastes cycles." But there is no session-level registry of what has been run. A simple `.planning/SESSION.md` tracking skill invocations and their outcomes within the current session would enable this check and prevent redundant re-runs.
