---
name: release-protocol
description: |
  Produces RELEASE.md — the deployment runbook, approval gates, rollback procedures, and success monitoring window. Every deployment is reversible and observable.

  <example>
  User says "how do we deploy this" or "create the release plan" — invoke release-protocol to produce RELEASE.md covering the numbered runbook, approval gates, rollback procedure, and monitoring window.
  </example>

  <example>
  After QA.md is confirmed, before first production deployment — invoke release-protocol to establish the complete deployment contract so the release-manager agent can execute it step-by-step.
  </example>
version: 1.0.0
---

# release-protocol

## What This Skill Produces

Produce `RELEASE.md` at `.planning/RELEASE.md` using the template at `references/assets/RELEASE.md.template`. The file contains four sections: Deployment Runbook, Approval Gates, Rollback Procedure, and Success Monitoring Window.

RELEASE.md is written before deployment, not during. Every step in the runbook is executable without prior knowledge of the system. Every approval gate specifies the evidence required to clear it. The rollback procedure is complete and tested before the deployment begins.

---

## Prerequisite Chain

Invoke release-protocol after:
1. QA.md is confirmed (qa-strategy skill) — provides the test results evidence required for the QA approval gate
2. DEV-SPEC.md is confirmed (dev-discipline skill) — provides the review completion evidence required for the review gate
3. All implementation phases complete — the runbook deploys a real artifact, not a placeholder

Do not produce RELEASE.md for code that has not passed QA. The approval gates reference QA.md pass criteria as required evidence. A RELEASE.md with blank approval gate evidence is not a release plan — it is a template.

---

## Section 1 — Deployment Runbook

The runbook is a numbered list of steps. Every step has four parts:

```
### Step N: <action>

**Command or link:** <exact command or URL, copy-paste ready>
**Expected output:** <measurable signal that this step succeeded>
**Failure signal:** <observable indicator that this step failed and rollback should be considered>
```

**Rules for runbook steps:**

- Write steps at the granularity where a new team member could execute them without asking anyone. "Deploy the service" is not a step. "Run `kubectl apply -f k8s/production.yaml` from the project root and wait for `deployment.apps/api configured` in the output" is a step.
- Every step has a defined expected output. The expected output is a measurable signal, not a judgment. "The service responds normally" is a judgment. "HTTP 200 from `curl https://api.example.com/health`" is a measurable signal.
- Every step has a failure signal. The failure signal is what tells the operator to stop and evaluate rollback rather than continuing. Include the observable symptom: exit code, error message text, absence of expected output.
- No implicit steps. Do not write "configure the environment as needed" — list every required environment variable, every configuration file change, every prerequisite command.
- Steps are sequential. If two steps can run in parallel, say so explicitly and explain the dependency. The default assumption is: complete Step N before beginning Step N+1.

**Minimum required runbook sections:**

1. Pre-deployment checks (verify prerequisites: artifact built and tagged, tests passed, environment healthy)
2. Deployment execution (the actual deployment commands)
3. Post-deployment smoke test (the minimal check that proves the deployment did not break the system)
4. Success confirmation (the signal that allows the operator to close the runbook and move to monitoring)

---

## Section 2 — Approval Gates

Approval gates are the checkpoints that must be cleared before deployment proceeds. Every gate specifies:

```
### Gate N: <gate name>

**Approver:** <who must approve — role, not person name>
**Evidence required:** <what artifact, result, or confirmation is required>
**Evidence location:** <where to find or record the evidence>
**CLEARED looks like:** <the observable state that means this gate is passed>
**BLOCKED looks like:** <the observable state that means deployment cannot proceed>
```

**Minimum required gates:**

**Gate 1: QA Pass**
Evidence: The test suite ran and all tests passed. The evidence is the test run output at the path specified in QA.md. For automated CI: the CI run link and a green status. For manual testing: a signed-off UAT checklist.

**Gate 2: Human Approval**
Evidence: At least one human reviewer who is not the author of the code has reviewed the changes and approved the deployment. The approver records their name and timestamp. Self-approval is never acceptable — one person cannot approve their own deployment to production.

Additional gates as required by the project:
- Dependency audit (no unreviewed new dependencies)
- Security scan pass (for systems handling financial or health data)
- Database migration review (when the deployment includes schema changes)
- Stakeholder sign-off (for user-visible feature launches)

**A gate with no defined evidence requirement is not a gate.** It is a wish. Every gate must specify exactly what evidence satisfies it and where that evidence is recorded.

---

## Section 3 — Rollback Procedure

Write the rollback procedure before deployment begins. A rollback procedure written after something goes wrong is written by someone under pressure with incomplete information. Write it when you are calm and informed.

```
### Rollback Trigger

**Observable signal:** <the specific, measurable condition that means "execute rollback now">
**Who decides:** <the role responsible for calling the rollback>
**Time threshold:** <how long to observe before declaring the deployment failed>
```

The rollback trigger is non-optional. Every deployment must define the observable signal that means "this deployment has failed and we must reverse it." A deployment without a defined rollback trigger is a deployment without an exit path.

**Rollback trigger examples:**

- "Error rate exceeds 0.5% for 2 consecutive minutes in the post-deployment monitoring window"
- "Smoke test exits with non-zero code after deployment completes"
- "p99 latency exceeds 3x pre-deployment baseline for 5 minutes"
- "Any CRITICAL error in the application log within 10 minutes of deployment"

The trigger must be specific enough that two different people evaluating the same monitoring dashboard would reach the same decision.

**Rollback steps:**

Format identically to the deployment runbook — numbered steps, each with command, expected output, and failure signal.

```
### Rollback Step N: <action>

**Command or link:** <exact command>
**Expected output:** <signal that rollback step succeeded>
**Failure signal:** <signal that rollback step itself failed — escalate immediately>
```

**Rollback types by deployment method:**

- **Container/image deploy:** Revert to the previous image tag. Command: `kubectl set image deployment/api api=registry/api:<previous-tag>`.
- **Feature flag deploy:** Toggle the flag off. Command: your feature flag service's disable command.
- **Database migration:** Execute the down migration. Document the down migration commands explicitly — do not reference "run the down migration" without the command.
- **DNS/load balancer cutover:** Update the DNS record or LB target to the previous endpoint.
- **Package deploy:** Reinstall the previous version. Command: `pip install app==<previous-version>` or equivalent.

Document which rollback type applies to this deployment in RELEASE.md. If the deployment touches multiple systems (e.g., code deploy + database migration), document rollback order explicitly — usually reverse order of deployment.

**Test the rollback before deploying.** In a staging environment, execute the deployment and then the rollback. If the rollback cannot be tested, document why and what the alternative recovery is.

---

## Section 4 — Success Monitoring Window

Define how long to observe the system after deployment before declaring it stable.

```
### Monitoring Window

**Duration:** <how long — expressed in minutes or hours>
**On-call contact:** <who is available during the window>
**Metrics to watch:** <the specific metrics and where to view them>
**Alert thresholds:** <the values that trigger escalation during the window>
**Stable signal:** <the observable state at the end of the window that means the deployment is confirmed stable>
```

**"A few hours" is not a duration.** Specify a number. For most web services, 30 minutes is the minimum. For high-traffic or high-stakes systems (payments, authentication, data pipelines), 2–4 hours is more appropriate.

**Metrics to watch during the window:**

At minimum: error rate, p99 latency, and any business metric directly affected by the deployment (e.g., if the deployment touches the payment flow, watch payment success rate).

The monitoring window uses tighter thresholds than the steady-state alert thresholds. A post-deployment error rate increase of 0.5% warrants immediate attention even if the steady-state alert threshold is 1%.

**Stable signal:**

Define the observable state that lets the on-call person close the monitoring window and go home. Example: "Error rate below 0.1% for 30 consecutive minutes. p99 latency within 10% of pre-deployment baseline. Zero CRITICAL log events."

---

## prereq-chain Integration

The release-protocol's final gate — Gate 2, Human Approval — is the Confirmation criteria for the project's deployment phase. When the release-manager agent records Gate 2 as CLEARED, the deployment phase is confirmed complete.

Reference the RELEASE.md approval gate record as the evidence artifact for deployment phase completion.

---

## Common Mistakes

**Runbook that skips "obvious" steps.** Every step that seems obvious to the person who wrote it is a step that will cause a 20-minute halt when someone else executes it and finds it is not documented. Write it down.

**Rollback procedure written post-incident.** Post-incident rollback procedures are written by people under pressure with an actively broken system. They contain errors. Write the rollback procedure during calm planning, verify it in staging, and then execute it in production if needed.

**Monitoring window of "a few hours."** Unmeasurable. "Monitor until things look stable" is not a protocol — it is an instruction to trust intuition. The monitoring window ends when a defined measurable condition is met.

**One person approving their own deployment.** A single point of failure for both the code quality and the deployment decision. Gate 2 requires a second human. If the team is truly one person, establish an async review process with a 24-hour window before production deployment.

---

## Worked Example: release-protocol for the Time-Tracking CLI

The following is an abbreviated RELEASE.md for the time-tracking CLI (Python CLI tool, no server infrastructure, deployed as a pip package).

```markdown
# RELEASE.md — time-tracker v1.0.0

## Deployment Runbook

### Step 1: Verify test suite passes

**Command:** `pytest --tb=short -q`
**Expected output:** `N passed in X.XXs` with no failures or errors.
**Failure signal:** Any line containing `FAILED` or `ERROR`.

### Step 2: Build the distribution package

**Command:** `python -m build`
**Expected output:** `dist/time_tracker-1.0.0.tar.gz` and
  `dist/time_tracker-1.0.0-py3-none-any.whl` created in `dist/`.
**Failure signal:** Non-zero exit code or missing files in `dist/`.

### Step 3: Upload to PyPI

**Command:** `twine upload dist/*`
**Expected output:** `View at: https://pypi.org/project/time-tracker/1.0.0/`
**Failure signal:** Any line containing `HTTPError` or `InvalidDistribution`.

### Step 4: Verify installation from PyPI

**Command:** `pip install time-tracker==1.0.0 --dry-run`
**Expected output:** `Would install time-tracker-1.0.0`
**Failure signal:** `ERROR: No matching distribution found`.

### Step 5: Smoke test on fresh install

**Command:** `pip install time-tracker==1.0.0 && timecli --version`
**Expected output:** `time-tracker 1.0.0`
**Failure signal:** Non-zero exit code or version mismatch.

## Approval Gates

### Gate 1: QA Pass
**Evidence required:** pytest output showing 0 failures.
**CLEARED:** pytest exits 0. Screenshot or log saved to `qa/run-YYYY-MM-DD.txt`.

### Gate 2: Human Approval
**Evidence required:** Second team member has reviewed the diff and signed off.
**CLEARED:** Reviewer records name and date in `qa/release-approval.txt`.

## Rollback Procedure

### Rollback Trigger

**Observable signal:** `timecli --version` exits non-zero OR returns a version
  other than 1.0.0 after smoke test.
**Who decides:** Deployer.
**Time threshold:** Immediate — smoke test failure in Step 5 triggers rollback.

### Rollback Step 1: Yank the broken release

**Command:** `twine upload --skip-existing` is not used. Contact PyPI support to
  yank 1.0.0: https://pypi.org/help/#yanking
**Expected output:** Version 1.0.0 marked as yanked on PyPI.
**Failure signal:** PyPI yanking unavailable — escalate to team lead.

## Success Monitoring Window

**Duration:** 24 hours post-release (no live server; monitoring is passive)
**Stable signal:** No error reports from early adopters in GitHub Issues within 24h.
```

---

## Writing the Artifact

When all four sections are complete, write the artifact to disk:

- **Path:** `.planning/RELEASE.md` at the project root
- Confirm with: `RELEASE.md written to .planning/RELEASE.md`

## References

- [`references/runbook-patterns.md`](references/runbook-patterns.md) — Runbook step format, how to write measurable "expected output" signals, common runbook anti-patterns, and 3 runbook examples at different complexity levels.
- [`references/rollback-strategies.md`](references/rollback-strategies.md) — Rollback strategy selection by deployment type, rollback trigger definition patterns, and how to test the rollback before deploying.
- [`references/assets/RELEASE.md.template`](references/assets/RELEASE.md.template) — Fill-in-the-blank RELEASE.md with all four sections and inline guidance comments.
