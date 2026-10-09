# comfy-templates-standby — cold-standby org runner

COLD STANDBY for the weekly comfy-templates data refresh workload.

- **Primary (healthy, active):**
  [comfy-backend/comfy-templates-runner](https://github.com/comfy-backend/comfy-templates-runner)
  — Wednesday 16:00 UTC + Thursday 17:07 UTC backstop (W18; the Netlify daily lane is primary), verified green E2E
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

SAFE ORDERING (2026-09-29, W2-c P1 fix): validate the standby BEFORE
trusting it. The promote used to start with "disable the primary" — if
the standby's `GH_PAT` had quietly rotted (rotations must hit BOTH
repos; the 2026-09-21 incident class), you would be left with NOTHING
green. The standby workflow is alert-enabled since 2026-09-29 (mirrors
the primary: failure → `data-refresh-alert` issue + @trinitylivy), so a
bad promote is at least LOUD now — but still validate first:

```bash
PAT=<github pat>
# 0. (preflight) confirm the standby's GH_PAT is fresh — the secrets
#    must have been rotated in lockstep (see Secrets below). If the last
#    rotation only hit the primary, rotate BOTH before promoting.
# 1. enable THIS standby (primary still active — overlap is safe, the
#    commit is race-safe rebase+retry, never force)
curl -X PUT -H "Authorization: token $PAT" \
  https://api.github.com/repos/comfyui-catalog/comfy-templates-standby/actions/workflows/refresh.yml/enable
# 2. validation run — REQUIRE GREEN before proceeding (a red run opens
#    a data-refresh-alert issue; fix that first, do NOT disable the
#    primary while the standby is unproven)
curl -X POST -H "Authorization: token $PAT" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/comfyui-catalog/comfy-templates-standby/actions/workflows/refresh.yml/dispatches \
  -d '{"ref":"main"}'
# 3. only after a green validation run: disable the (misbehaving) primary
curl -X PUT -H "Authorization: token $PAT" \
  https://api.github.com/repos/comfy-backend/comfy-templates-runner/actions/workflows/refresh.yml/disable
```

Demotion is the mirror image (validate the primary with a dispatch,
then disable the standby). The workload itself is stateless — everything
it touches lives in the private repo, and its data commit is race-safe
(rebase + retry, never force), so a promote/demote pair can even overlap
without data risk.

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
commentary; this file mirrors it — only the workflow name, the banner
and RUNNER_REPO may differ): checkout the private `trinitylivy/comfy-templates` → run
the 8-step upstream refresh pipeline (100% Python stdlib) → mirror the
manifest-listed files to `public/data` → 10-check structural audit
(A–J, incl. the upstream schema pin) → app consumption contract test
(vitest) → race-safe data commit to the private repo's main → post-push
prod verification (Vercel deploy polled until the new built_at is live)
→ failure alerting (`data-refresh-alert` issues, cc @trinitylivy) +
failure-log artifacts. Nothing sensitive is ever committed here.
