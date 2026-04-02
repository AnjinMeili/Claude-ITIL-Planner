# Rollback Strategies Reference

Rollback strategy selection by deployment type, rollback trigger definition patterns, and how to test the rollback before deploying.

---

## Rollback Strategy Selection

The correct rollback strategy depends on what was deployed. Select the strategy that matches the deployment type. For deployments that touch multiple systems (e.g., code + database), combine strategies and document the rollback order explicitly (almost always reverse deployment order).

---

### Strategy 1: Container Image Revert

**When to use:** The deployment changed a Docker image on a container orchestration platform (Kubernetes, ECS, Cloud Run).

**How it works:** Update the deployment to reference the previous image tag. The orchestrator replaces running containers with the previous version.

**Command (Kubernetes):**
```bash
# Find the previous image tag
kubectl rollout history deployment/<name> -n <namespace>

# Revert to previous revision
kubectl rollout undo deployment/<name> -n <namespace>

# Or specify a revision number
kubectl rollout undo deployment/<name> --to-revision=<N> -n <namespace>

# Verify rollback completed
kubectl rollout status deployment/<name> -n <namespace>
```

**Expected output:** `deployment.apps/<name> successfully rolled out`

**Rollback time:** 1–5 minutes depending on image pull time.

**Limitation:** Does not revert database schema changes. If the deployment included a migration, the old code may be incompatible with the new schema. Plan for this: either the migration must be backward-compatible (old code can run against new schema), or the migration rollback must happen first.

---

### Strategy 2: Feature Flag Toggle

**When to use:** The new behavior is gated behind a feature flag. The deployment is already in production; the flag gates who sees the new behavior.

**How it works:** Toggle the flag off. The new code path becomes unreachable; the system behaves as before the flag was enabled.

**Command (generic — substitute your feature flag service):**
```bash
# LaunchDarkly CLI
ld-cli feature-flag update --key <flag-key> --environment production --enabled false

# Unleash
curl -X POST https://unleash.example.com/api/admin/features/<flag-name>/environments/production/off \
  -H "Authorization: Bearer $UNLEASH_TOKEN"

# Custom config file
jq '.flags.<flag_name> = false' config/flags.production.json > tmp.json && mv tmp.json config/flags.production.json
git commit -m "chore: disable <flag-name> — rollback" && git push
```

**Rollback time:** Near-instant for in-memory flag evaluation. Up to 60 seconds for config-file-based flags with polling intervals.

**Advantage:** No redeployment required. Fastest available rollback. Allows partial rollback (disable for a user segment before disabling entirely).

**Requirement:** Feature flag infrastructure must be in place before the deployment begins. Retrofitting flags into a deployment that is already broken is not a rollback strategy.

---

### Strategy 3: Database Migration Rollback

**When to use:** The deployment included a schema change (migration) and the migration must be reversed.

**How it works:** Execute the down migration. The schema reverts to its pre-deployment state. Application code must be rolled back first (or simultaneously) to avoid running new code against the old schema.

**Order of operations:**
1. Roll back application code (Strategy 1 or 2) — immediately
2. Execute down migration — after code rollback confirms healthy

**Command (Alembic — Python):**
```bash
# View migration history
alembic history

# Roll back to the previous revision
alembic downgrade -1

# Roll back to a specific revision
alembic downgrade <revision_id>

# Verify current schema version
alembic current
```

**Command (Flyway — Java):**
```bash
# Repair and undo the last migration
flyway undo
```

**Command (Django):**
```bash
python manage.py migrate <app_name> <previous_migration_number>
```

**Critical requirement:** Down migrations must be written before deployment and tested in staging. An up-only migration strategy makes schema rollback impossible. If down migrations are not written, the only recovery from a bad migration is a database restore from backup.

**Testing the down migration:** Execute `up → down → up` in the staging environment before deploying to production. If the round-trip produces a different schema than two sequential ups, the down migration has a defect.

---

### Strategy 4: DNS Failover / Load Balancer Cutover

**When to use:** The deployment switched traffic to a new environment or a new set of instances, and the old environment is still running.

**How it works:** Update the DNS record or load balancer target to route traffic back to the previous environment.

**Command (Route 53 — AWS CLI):**
```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id <ZONE_ID> \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.example.com.",
        "Type": "CNAME",
        "TTL": 60,
        "ResourceRecords": [{"Value": "<previous-endpoint>"}]
      }
    }]
  }'
```

**Command (nginx upstream swap):**
```bash
# Update upstream config to point to previous servers
# Reload nginx without dropping connections
nginx -t && nginx -s reload
```

**Rollback time:** DNS TTL-dependent. If TTL is 300 seconds, full propagation takes up to 5 minutes. Set TTL to 60 seconds before deploying when a fast rollback window is important.

**Advantage:** The old environment is still running and healthy. Rollback is a traffic redirect, not a redeployment.

**Requirement:** The old environment must remain running and healthy until the monitoring window closes. Do not terminate the previous environment immediately after deploying.

---

### Strategy 5: Package Version Revert

**When to use:** The deployment was a package version bump (pip, npm, gem, etc.) on a server or in a running environment.

**Python:**
```bash
pip install <package-name>==<previous-version>
# Or from requirements.txt with previous version pinned
pip install -r requirements-previous.txt
# Restart the application process
systemctl restart <service-name>
```

**Node.js:**
```bash
npm install <package-name>@<previous-version>
# Restart
pm2 restart <app-name>
```

**Rollback time:** Depends on package download speed. 1–3 minutes typically.

**Limitation:** Does not work if the new version introduced a file format change or data migration. Check the changelog before relying on this strategy.

---

## Rollback Trigger Definition Patterns

A rollback trigger is the observable signal that means "execute rollback now." Define it before deployment, in terms that are unambiguous to anyone monitoring the system.

### Pattern 1: Error Rate Threshold

```
TRIGGER if: error_rate > X% over N minutes in the post-deployment monitoring window
```

Choose X based on the system's normal error rate:
- For a system with ~0% baseline errors: trigger at 0.1% for 2 minutes
- For a system with ~0.1% baseline errors: trigger at 0.5% for 3 minutes
- Never trigger above 1% — at that rate, users are actively affected

### Pattern 2: Smoke Test Failure

```
TRIGGER if: post-deployment smoke test exits with non-zero code
```

Use for: CLI tools, API deployments where a test script can be run immediately after deploy. The cleanest trigger — binary pass/fail, no interpretation required.

### Pattern 3: Latency Threshold

```
TRIGGER if: p99 latency > Nx pre-deployment baseline for Y minutes
```

Requires knowing the pre-deployment baseline. Measure it immediately before deploying; record it in RELEASE.md.

### Pattern 4: Business Metric Degradation

```
TRIGGER if: payment_success_rate < X% for N consecutive minutes
```

Use for: deployments to business-critical paths where a code error would degrade a key metric in an observable way. Requires near-real-time business metric dashboards.

### Pattern 5: Explicit Symptom

```
TRIGGER if: any log line matching the pattern "FATAL: unable to connect to payment provider"
  appears within 10 minutes of deployment
```

Use when the deployment touches a specific integration and a specific failure mode is the likely failure. More specific than an error rate threshold; fires faster when the exact failure occurs.

---

## How to Test the Rollback Before Deploying

Testing the rollback is not optional. A rollback procedure that has never been executed is a rollback procedure with unknown defects.

**Staging environment rollback test protocol:**

1. Deploy the new version to staging
2. Verify the deployment is healthy in staging
3. Execute the rollback procedure in staging
4. Verify the system returns to the previous state (schema, behavior, response)
5. If any rollback step fails: fix it before deploying to production

**Document the rollback test result in RELEASE.md:**

```markdown
## Rollback Verification

**Verified in:** staging
**Verified by:** <name>
**Date verified:** YYYY-MM-DD
**Result:** PASSED / FAILED
**Notes:** <any issues found and how they were resolved>
```

**For database migration rollbacks specifically:**

Execute `up → down → up` in staging and verify the schema after each step. Also verify that data written after the up migration survives the down migration without data loss — if the down migration drops a column, any data in that column is lost.

If the down migration is destructive (data loss), document this explicitly in RELEASE.md and reconsider whether the rollback strategy should be "restore from backup" instead of "execute down migration."
