# Runbook Patterns Reference

Runbook step format, measurable "expected output" guidance, anti-patterns, and 3 examples at different complexity levels.

---

## The Runbook Step Format

Every runbook step follows the same structure:

```
### Step N: <verb-first action description>

**Command or link:** <exact command or URL>
**Expected output:** <measurable signal of success>
**Failure signal:** <observable indicator that this step failed>
```

**Action description:** Verb-first, specific. "Run the smoke test" not "Smoke test".

**Command or link:** Copy-paste ready. Include the full command with all flags. Include the working directory if it matters. If the step is a UI action, provide the exact URL and the sequence of clicks.

**Expected output:** The observable evidence that the step completed successfully. Write a measurable signal, not a judgment.

**Failure signal:** What the operator sees when the step has gone wrong. Be specific enough that the operator does not need to interpret — they either see this signal or they do not.

---

## Writing Measurable "Expected Output"

The most common runbook failure is vague expected output. The operator executes the step, something appears on the screen, and they cannot tell whether it succeeded.

**Vague (judgment required):**
- "The service starts successfully"
- "The deployment completes"
- "The health check passes"
- "Output looks normal"

**Measurable (no judgment required):**
- "The process exits with code 0 and the final line of output is `INFO: Server started on port 8080`"
- "`kubectl rollout status deployment/api` outputs `deployment.apps/api successfully rolled out`"
- "`curl -s -o /dev/null -w '%{http_code}' https://api.example.com/health` returns `200`"
- "The database migration log ends with `Applying 0005_add_user_index... OK`"

**How to make output measurable:**

1. Specify the exact string to look for (grep-able output)
2. Specify the exit code (`exits with code 0`)
3. Specify the absence of an error indicator (`no lines containing ERROR or FAILED`)
4. Specify a measurable state (`row count in the users table increases by 1`)

---

## Common Anti-Patterns

### Missing rollback steps

A runbook that says "if step 4 fails, assess the situation" is not a runbook — it is a flowchart with a missing branch. Every runbook must reference the rollback procedure or include rollback steps inline. The failure signal for every step answers "what do I do now?" — and the answer is either "continue" or "execute rollback."

### Implicit steps

"Configure the environment variables" is not a step. List every required variable, its expected format, and where to set it. "Ensure the database is running" is not a step. Provide the command to check and the expected output.

The test for an implicit step: could a new engineer who has never worked on this project execute this step without asking anyone? If not, it is implicit.

### Single-person approval

A runbook that records only the deployer's name as evidence of approval has a single point of failure: the same person who wrote the code is also the person verifying it is safe to deploy. Gate 2 requires a second human.

### Steps that cannot be verified

"Wait for the deployment to complete" without a command to check completion status. Replace with: "Run `kubectl rollout status deployment/api --timeout=5m`. Wait for `successfully rolled out`. If the command times out, execute rollback."

### Undated runbooks

Runbooks drift. A runbook written 6 months ago may reference commands, endpoints, or tools that have changed. Include the version of the software being deployed and the date the runbook was written. Review and update the runbook with each deployment.

---

## Example 1: Simple Python CLI Release (Low Complexity)

```markdown
## Deployment Runbook — time-tracker v1.2.0

### Step 1: Confirm test suite passes

**Command:** `pytest --tb=short -q`
**Expected output:** `N passed` with no failures. Exit code 0.
**Failure signal:** Any FAILED or ERROR in output. Exit code non-zero.

### Step 2: Bump version

**Command:** Edit `pyproject.toml`, set `version = "1.2.0"`. Commit:
  `git commit -m "chore: bump version to 1.2.0"`
**Expected output:** `[main <sha>] chore: bump version to 1.2.0`
**Failure signal:** git exits non-zero.

### Step 3: Build distribution

**Command:** `python -m build --outdir dist/`
**Expected output:** `dist/time_tracker-1.2.0.tar.gz` and
  `dist/time_tracker-1.2.0-py3-none-any.whl` present in `dist/`.
**Failure signal:** Non-zero exit or missing files.

### Step 4: Upload to PyPI

**Command:** `twine upload dist/time_tracker-1.2.0*`
**Expected output:** `View at: https://pypi.org/project/time-tracker/1.2.0/`
**Failure signal:** `HTTPError` or `InvalidDistribution` in output.

### Step 5: Smoke test

**Command:** `pip install --upgrade time-tracker==1.2.0 && timecli --version`
**Expected output:** `time-tracker 1.2.0`
**Failure signal:** Version mismatch or non-zero exit.
```

---

## Example 2: Node.js API Deployment via Docker + Kubernetes (Medium Complexity)

```markdown
## Deployment Runbook — orders-api v3.4.1

### Step 1: Verify CI pipeline passed

**Link:** https://ci.example.com/projects/orders-api/pipelines
**Expected output:** Pipeline for commit <SHA> shows green status on all stages.
**Failure signal:** Any stage shows red/failed. Do not proceed.

### Step 2: Pull and tag the release image

**Command:**
  `docker pull registry.example.com/orders-api:sha-<COMMIT_SHA>`
  `docker tag registry.example.com/orders-api:sha-<COMMIT_SHA> registry.example.com/orders-api:v3.4.1`
  `docker push registry.example.com/orders-api:v3.4.1`
**Expected output:** `digest: sha256:...` on push.
**Failure signal:** Non-zero exit or `unauthorized` error.

### Step 3: Update Kubernetes deployment

**Command:**
  `kubectl set image deployment/orders-api orders-api=registry.example.com/orders-api:v3.4.1 -n production`
  `kubectl rollout status deployment/orders-api -n production --timeout=5m`
**Expected output:** `deployment.apps/orders-api image updated`
  then `Waiting for deployment "orders-api" rollout to finish: 1 out of 3 new replicas have been updated...`
  then `deployment.apps/orders-api successfully rolled out`
**Failure signal:** `error:` in output or command times out at 5m.

### Step 4: Health check

**Command:**
  `curl -s -o /dev/null -w '%{http_code}' https://api.example.com/health`
**Expected output:** `200`
**Failure signal:** Any code other than `200`. If 5xx: execute rollback immediately.

### Step 5: Smoke test key endpoints

**Command:** `./scripts/smoke-test.sh production`
**Expected output:** `All smoke tests passed (8/8)`
**Failure signal:** Any `FAIL` in output or non-zero exit.
```

---

## Example 3: Database Migration Deployment (High Complexity — Migration Included)

```markdown
## Deployment Runbook — billing v2.1.0 (includes schema migration)

### Step 1: Back up the production database

**Command:** `pg_dump $DATABASE_URL -Fc -f backups/billing-pre-v2.1.0-$(date +%Y%m%d%H%M%S).dump`
**Expected output:** File created in `backups/` with size >0 bytes.
  Verify: `ls -lh backups/billing-pre-v2.1.0-*.dump`
**Failure signal:** Non-zero exit or file size 0.
**Note:** Do not proceed without a successful backup. This step protects against
  migration failure.

### Step 2: Enable maintenance mode

**Command:** `./scripts/maintenance on`
**Expected output:** `https://billing.example.com` returns HTTP 503 with
  maintenance page. Verify: `curl -s -o /dev/null -w '%{http_code}' https://billing.example.com`
  returns `503`.
**Failure signal:** Returns 200 or other non-503 code.

### Step 3: Run database migration

**Command:** `alembic upgrade head`
**Expected output:** Final line: `INFO  [alembic.runtime.migration] Running upgrade
  <prev_rev> -> <new_rev>, Add invoice_currency column`
**Failure signal:** Any line containing `ERROR` or `FAILED`. If migration fails:
  execute rollback Step 1 (restore from backup) before any other recovery action.

### Step 4: Deploy application

**Command:**
  `kubectl set image deployment/billing billing=registry.example.com/billing:v2.1.0 -n production`
  `kubectl rollout status deployment/billing -n production --timeout=10m`
**Expected output:** `deployment.apps/billing successfully rolled out`
**Failure signal:** Rollout timeout or error. Execute rollback.

### Step 5: Disable maintenance mode

**Command:** `./scripts/maintenance off`
**Expected output:** `https://billing.example.com` returns HTTP 200.
**Failure signal:** Still returns 503 after 30 seconds.

### Step 6: Smoke test

**Command:** `./scripts/smoke-test.sh production`
**Expected output:** `All smoke tests passed (12/12)`
**Failure signal:** Any failure. Execute rollback.
```
