# Rollback Runbook — Vision Moderation Service

**Owner:** ML Platform on-call | **Updated:** see git log

---

## When to Roll Back

Trigger rollback immediately if **any** of these are true:

- [ ] Error rate > 1% sustained for 2+ minutes (SLO breach — `AvailabilityBurnFast` firing)
- [ ] p99 latency > 1s sustained for 5+ minutes (`LatencyP99High` firing)
- [ ] Model version mismatch on any replica for 5+ minutes (`ModelVersionMismatch` firing)
- [ ] Human reviewer reports classification quality collapse (>5% label flip rate vs baseline)
- [ ] Canary verify script exits non-zero during deploy

**Do not wait for the slow-burn alert.** If two fast signals fire together, roll back now.

---

## How to Roll Back

### Option A — GitHub Actions (preferred)

```
gh workflow run deploy-model.yml \
  --ref main \
  -f target_env=production \
  -f image_tag=<LAST_KNOWN_GOOD_SHA>
```

Or via UI: **Actions → Deploy Vision Moderation Model → Run workflow**

### Option B — Direct script (if Actions is unavailable)

```bash
export DEPLOY_ENV=production
export IMAGE_TAG=<LAST_KNOWN_GOOD_SHA>
export CANARY_WEIGHT=100
bash ./scripts/rollback.sh
```

`LAST_KNOWN_GOOD_SHA` → find in `#ml-deploys` Slack channel or the last green deploy in Actions.

---

## What to Verify

After rollback completes, confirm all green within **5 minutes**:

- [ ] `AvailabilityBurnFast` and `LatencyP99High` resolved in Alertmanager
- [ ] Error rate < 0.1% on [SLO dashboard](https://grafana.example.com/d/vision-mod-slo)
- [ ] p99 latency < 500ms on [Latency dashboard](https://grafana.example.com/d/vision-mod-latency)
- [ ] All replicas show expected `model_version` label on [Version dashboard](https://grafana.example.com/d/vision-mod-versions)
- [ ] No `ModelVersionMismatch` alert pending

---

## Who to Notify

| When | Where | Who |
|---|---|---|
| Rollback initiated | `#ml-incidents` Slack | `@ml-platform-oncall` |
| Rollback complete / failed | `#ml-incidents` Slack | `@ml-platform-oncall` + `@eng-oncall` |
| Customer impact confirmed | `#incidents` Slack | Incident commander |
| Post-mortem scheduled | Linear ticket (auto) | Team lead |

---

## What NOT To Do

- ❌ **Do not roll forward** to a hotfix before root cause is identified
- ❌ **Do not restart individual pods** to "partially fix" — version skew makes debugging harder
- ❌ **Do not silence alerts** without completing the rollback
- ❌ **Do not roll back the database** or feature store — model rollback is stateless; data rollback is a separate, much larger decision requiring staff-level sign-off
- ❌ **Do not skip the verification checklist** even if the graphs look fine — wait the full 5 minutes

---

## When to Roll Forward

Only re-deploy the new version when **all** of these are true:

- [ ] Root cause is identified and documented in the Linear ticket
- [ ] Fix is committed, reviewed, and merged
- [ ] Unit + contract tests pass on the fix branch
- [ ] Staging smoke tests pass
- [ ] On-call lead has approved the re-deploy in the `#ml-incidents` thread
