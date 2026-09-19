# comfy-templates-standby — cold-standby org runner

COLD STANDBY for the weekly comfy-templates data refresh workload.

- **Primary (healthy, active):**
  [comfy-backend/comfy-templates-runner](https://github.com/comfy-backend/comfy-templates-runner)
  — Monday 16:00 UTC + Tuesday 17:07 UTC backstop, verified green E2E
  since 2026-09-09.
- **This repo (dormant):** identical workload
  (`.github/workflows/refresh.yml`), GH_PAT secret configured, workflow
  **DISABLED** so the crons never double-fire while the primary is
  healthy. Last smoke-tested E2E: see `state/last-run.json`.

## Why a standby

The `trinitylivy` personal account is startup_failure-blocked
account-wide for Actions (comfy-templates `KNOWN_ISSUES #27`), so all
scheduled compute must live in healthy orgs. Public repos get free
unlimited Actions minutes. The `comfyui-catalog` org (created
2026-09-19) is the second healthy island: this standby covers primary
failover, and the org is also the designated home for any future
private-repo workloads or extra compute/cron needs (free org plan
includes 2,000 private-repo Actions minutes/month).

## Promote / demote runbook (failover)

```bash
PAT=<github pat>
# 1. disable the PRIMARY (if it is misbehaving)
curl -X PUT -H "Authorization: token $PAT" \
  https://api.github.com/repos/comfy-backend/comfy-templates-runner/actions/workflows/refresh.yml/disable
# 2. enable THIS standby
curl -X PUT -H "Authorization: token $PAT" \
  https://api.github.com/repos/comfyui-catalog/comfy-templates-standby/actions/workflows/refresh.yml/enable
# 3. optional: immediate validation run
curl -X POST -H "Authorization: token $PAT" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/comfyui-catalog/comfy-templates-standby/actions/workflows/refresh.yml/dispatches \
  -d '{"ref":"main"}'
```

Demotion is the mirror image (enable primary, disable standby). The
workload itself is stateless — everything it touches lives in the
private repo, and its data commit is race-safe (rebase + retry, never
force), so a promote/demote pair can even overlap without data risk.

## Secrets

`GH_PAT` — encrypted Actions secret, same value as the primary runner's.
Rotate on BOTH repos with the tool in the main repo:

```bash
SECRET_VALUE=<new pat> python3 scripts/org-runner/set-secret.py \
    comfy-backend/comfy-templates-runner
SECRET_VALUE=<new pat> python3 scripts/org-runner/set-secret.py \
    comfyui-catalog/comfy-templates-standby
```

## What the workload does

Identical to the primary (see that repo's workflow file for full
commentary): checkout the private `trinitylivy/comfy-templates` → run
the 8-step upstream refresh pipeline (100% Python stdlib) → mirror the
manifest-listed files to `public/data` → 7-check structural audit →
race-safe data commit to the private repo's main → Vercel auto-deploys.
Nothing sensitive is ever committed here.
