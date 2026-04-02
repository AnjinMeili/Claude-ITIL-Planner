# Risk and Debt Patterns — Reference Guide

## RISKS.md

### Entry Format

```
## RISK-NNN: [Risk Description]

- **Date identified:** YYYY-MM-DD
- **Likelihood:** High | Medium | Low
- **Impact:** High | Medium | Low
- **Mitigation strategy:** [Specific action — not "monitor closely"]
- **Status:** Open | Mitigated | Accepted | Closed
```

Write the risk description as "X could happen, causing Y." Example: "SQLite schema migrations could fail during CLI upgrades, causing users to lose access to their entry history."

### Likelihood/Impact Matrix with Response Strategies

| Likelihood | Impact | Default Strategy | Action |
|---|---|---|---|
| High | High | **Mitigate** | Block progress until mitigation is in place |
| High | Medium | **Mitigate** | Address before the affected component ships |
| High | Low | **Accept** | Define a monitoring trigger; log in RISKS.md |
| Medium | High | **Mitigate proactively** | Add to DEBT.md if mitigation deferred |
| Medium | Medium | **Mitigate or Accept** | Evaluate cost of mitigation vs cost of occurrence |
| Medium | Low | **Accept** | Log and monitor |
| Low | High | **Mitigate proactively** | Assign to a specific owner; date-targeted |
| Low | Low | **Accept** | Log and move on |

### The Four Response Strategies

**Mitigate:** Take a specific action that reduces likelihood or impact before the risk materializes. A mitigation strategy must describe a concrete action — not "monitor closely," not "be careful," not "test thoroughly." Example: "Adopt Alembic from v0.1; include a schema version check at CLI startup; write a migration script alongside every schema-change PR."

**Accept:** Acknowledge the risk, decide the cost of mitigation exceeds the expected cost of occurrence, and proceed. Accepted risks require a monitoring trigger — a specific signal that would escalate the risk back to Mitigate. Example: "Accept: migration failures are unlikely in v1 with a single user. Trigger: if any user reports a migration failure, escalate to Mitigate immediately before the next release."

**Transfer:** Assign the risk to a third party. This is most common with vendor SLAs, insurance, or external team ownership. Example: "Transfer: PyPI availability risk transferred to PyPI SLA. Monitor: PyPI status page."

**Avoid:** Change the approach to eliminate the risk entirely. Example: "Avoid: ship a migration-free flat schema for v1; eliminate migration risk by designing schema stability into the v1 contract."

### Writing Mitigation Strategies

Weak mitigation strategies fail because they describe intent rather than action:
- Weak: "Monitor the schema migration process."
- Weak: "Test migrations thoroughly."
- Weak: "Be careful about schema changes."

Strong mitigation strategies describe a specific, schedulable action:
- Strong: "Add `alembic` to project dependencies before the first schema-touching commit; require a migration script in every PR that touches `schema.py`; add a startup check that compares stored schema version to expected version and exits with a clear error if they differ."

### Closing a Risk

Update Status to `Closed` only with evidence. Acceptable evidence:
- The risky component shipped successfully through at least one production release.
- An external dependency was confirmed by the vendor or by a successful integration test.
- A spike resolved the technical uncertainty that created the risk.
- The scope that created the risk was removed from the project.

"It didn't happen" is not evidence. "We shipped v1 and no migration failures were reported across 50 installs, monitored for 30 days" is evidence.

---

## DEBT.md

### Entry Format

```
## DEBT-NNN: [What Was Deferred]

- **Date deferred:** YYYY-MM-DD
- **Why deferred:** [Acceptable reason — see below]
- **Impact of deferral:** [What breaks, degrades, or accumulates cost if this is never resolved]
- **Target resolution date:** YYYY-MM-DD or Before [milestone name]
- **Status:** Open | In Progress | Resolved
```

### Debt Classification

**Intentional shortcut:** A simpler approach chosen consciously to meet a deadline, with the known technical cost documented. This is legitimate debt. Example: "Hard-coded the database path for v1 rather than implementing XDG base directory spec. Impact: users cannot customize the storage location; multi-user environments are unsupported."

**External constraint:** A dependency, vendor limitation, or regulatory requirement outside the team's control. Example: "PyPI API does not support bulk metadata queries; batch stats feature deferred until PyPI adds bulk endpoint or we build a cache layer."

**Sequencing constraint:** This work can only be done after another component exists. Example: "Cross-device sync deferred until local storage layer is stable — sync requires a confirmed schema."

**Unintentional oversight:** Something that should have been done but was missed. Record it as debt, not a defect, only if it does not create user-visible failures. If it creates failures, treat it as a defect and track in the issue tracker.

**Bit rot:** Code or configuration that worked when written but has been made obsolete by external changes (library updates, platform changes, dependency deprecations). Example: "Setup.py-based packaging to be replaced with pyproject.toml before Python 3.14 removes distutils."

### Acceptable Deferral Reasons

A DEBT.md entry requires an acceptable reason. The test: could you defend this deferral to a new team member joining in six months? If not, it is not a legitimate deferral reason.

Acceptable:
- "v1 scope was bounded to offline-only; sync was explicitly out of scope per SCOPE.md, Decision DEC-003."
- "PyPI bulk API does not exist; deferred until available."
- "Cannot implement migration tooling before schema stabilizes in v1."

Not acceptable:
- "We'll do it later." (Not a reason — restate what constraint makes this the right time to defer.)
- "It's too hard right now." (Not a reason — identify the specific blocker and whether it resolves.)
- "Not a priority." (This is a consequence, not a reason — explain why other work takes priority and when that changes.)

### Debt That Is Never Acceptable

Do not record as DEBT.md entries — treat these as defects:

- Known security vulnerabilities with no mitigation in place
- Missing error handling on user-facing flows that causes silent data loss
- Removal of test coverage that was previously providing a safety net
- Bypassing validation on user input in paths that affect stored data

These are defects. Logging them as "debt" creates a false impression that deferral is an acceptable choice. Track defects in the issue tracker, not in DEBT.md.

### Target Dates Are Mandatory

Every DEBT.md entry must have a target resolution date. A debt entry without a target date is permanent debt — it will never be resolved because it was never scheduled.

If the exact date is unknown:
- Anchor to a milestone: `Before v2.0`
- Anchor to a condition: `Before first multi-user deployment`
- Anchor to an external event: `Before Python 3.14 release`

Never leave target date blank. If none of these anchors apply, escalate the discussion — debt with no resolution path is a scope decision that needs to be made explicitly.

### Resolving Debt

Update Status to `Resolved` only with evidence. Acceptable evidence:
- A commit or PR that implements the deferred work.
- A scope decision that removes the requirement the debt was serving.
- An external dependency that arrived and enabled the implementation.

Add a note below the Status line: `Resolved by: [commit hash or PR link or scope change reference] on YYYY-MM-DD`.

### Cross-Register Links

- If debt was caused by a decision, note: `Caused by: DEC-NNN`
- If debt creates a risk (deferral itself introduces risk), note: `Creates risk: RISK-NNN`
- If debt resolves when an assumption is validated, note: `Unblocked when: ASS-NNN confirmed`

## File Initialization

If RISKS.md does not exist:

```
# Risk Register

Tracks identified risks with likelihood, impact, and mitigation. Append only — do not edit existing entries except to update Status fields.

---
```

If DEBT.md does not exist:

```
# Debt Register

Tracks deferred work with required rationale and target resolution dates. Append only — do not edit existing entries except to update Status fields.

---
```
