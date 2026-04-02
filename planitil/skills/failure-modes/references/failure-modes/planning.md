# Planning Failure Modes

Eight named failure modes for plans, work breakdowns, and task sequences. Each failure mode has a symptom pattern and an explicit STOP condition. A STOP condition is a specific, observable signal — if present, halt and restart with the condition resolved.

---

## 1. Orphaned Goal

**Symptom:** Plan tasks complete but the original goal is not met. Individual steps pass their local criteria; the goal-level outcome is never verified.

**STOP condition:** No explicit goal-backward verification step exists in the plan — no step that checks whether the original goal has been achieved after all tasks complete.

---

## 2. Assumed Context

**Symptom:** The plan assumes information that the executor will not have. Steps that were clear during planning become ambiguous at execution time because the shared context is not written down.

**STOP condition:** Any plan step contains language such as "as discussed," "per the requirements," "as agreed," or "based on the earlier decision" — references to context not captured in the plan itself.

---

## 3. Missing Rollback

**Symptom:** A plan step makes a destructive or irreversible change with no documented way to undo it. When that step fails, recovery requires improvisation.

**STOP condition:** Any plan step that is destructive or irreversible (data deletion, schema migration, infrastructure teardown, external API mutation) has no rollback path documented alongside it.

---

## 4. Dependency Blindness

**Symptom:** Steps that depend on earlier outputs are not sequenced correctly. An executor reaches a step and finds the required input has not been produced yet.

**STOP condition:** Any plan step N references an output or artifact that is produced by a step M where M appears after N in the sequence.

---

## 5. Phantom Verification

**Symptom:** The plan includes verification steps that cannot be executed because they specify no method, tool, or pass/fail criteria. "Verify X works" is not a verification step.

**STOP condition:** Any verification step in the plan has no explicit pass/fail criteria — it cannot be determined with certainty whether the step passed or failed.

---

## 6. Scope Inflation

**Symptom:** The plan has grown beyond what `SCOPE.md` defined. Tasks that were not in the agreed scope appear in the plan, silently expanding the work.

**STOP condition:** Any plan task is not traceable to a success criterion in `SCOPE.md` — it cannot be pointed to a specific scope item that requires it.

---

## 7. Single Thread

**Symptom:** The plan sequences all work serially when some steps are independent and could run in parallel. This unnecessarily extends execution time and creates unnecessary sequential dependencies.

**STOP condition:** The plan contains more than five sequential steps with no parallel opportunities identified or flagged, in a domain where parallelism is safe (no shared mutable state between candidates).

---

## 8. Vague Done

**Symptom:** Completion criteria are subjective or unmeasurable. The plan cannot be declared done by a third party because "done" depends on judgment calls.

**STOP condition:** Any completion criterion uses language such as "done when it works," "done when it looks right," "done when it seems stable," or any other subjective/unobservable standard.
