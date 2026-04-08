---
name: deploy
description: "Deploy application code to production. Onboard / execute / check modes driven by deploy/deployment-config.md."
---

# /deploy - Application Deployment

For applications that ship code to running environments (containers, edge functions, VMs, k8s, mobile stores). Driven by `deploy/deployment-config.md`.

**NOT to be confused with `/release`** — `/release` publishes packages to artifact registries (PyPI, crates.io, npm). `/deploy` ships running code. If your repo's `deploy/deployment-config.md` declares `type: sdk`, this command redirects to `/release`.

## Mode Detection

```
/deploy            → Execute mode
/deploy --check    → Check mode (drift detection only, no deploy)
/deploy --onboard  → Onboard mode (create deployment-config.md)
```

### If `deploy/deployment-config.md` does NOT exist → Onboard Mode

Run the application deployment onboarding process. See `skills/10-deployment-git/application-deployment.md`.

1. **Detect platform** — read repo for clues: `Dockerfile`, `vercel.json`, `fly.toml`, `app.yaml`, `kubernetes/`, `azure-pipelines.yml`, `containerapps/`, `Procfile`, native binaries
2. **Ask the human** — what platform, what deploy command, what counts as "production code", how to query current deployed state, staging required?
3. **Research current best practices** — web search for platform-specific deploy patterns (Container Apps revisions, Fly machines, Cloud Run revisions, etc.). Do NOT rely on encoded knowledge — cloud platform CLIs change frequently.
4. **Create `deploy/deployment-config.md`** — see schema in `skills/10-deployment-git/application-deployment.md`
5. **STOP — present to human for review**

### If `deploy/deployment-config.md` declares `type: sdk` → Redirect

This repo is an SDK. Run `/release` instead — it handles version bumping, PyPI/registry publishing, and artifact validation.

### If `deploy/deployment-config.md` declares `type: application` → Execute Mode

Read the config and execute. **Print the 10-step DEPLOY CHECKLIST at the start of the response and check off boxes as each step passes. Do NOT report deploy as complete until every box is checked.** The full checklist text and per-step guidance lives in `rules/deploy-hygiene.md` Rule 8 and `skills/10-deployment-git/application-deployment.md`. If any step fails, say "DEPLOY FAILED AT STEP N: <reason>" — NOT "build succeeded, will redeploy soon".

#### Step 0: Pre-Deploy Verification

1. **Drift check** — run `deploy_check_command` from config. Compare current deployed commit/revision against `git rev-parse HEAD`.
2. **Production paths diff** — `git diff <last_deployed_commit> HEAD -- <production_paths>` to confirm what's actually being shipped.
3. **Present diff summary to human and STOP for approval** if any of: untested changes, schema migrations, secret/config changes, breaking API changes.

#### Step 1: Pre-Deploy Gates

Run the gate commands from config (typically: tests, lint, security scan). Block on failure unless `--skip-gates` is explicitly set with a documented reason.

**Why each gate exists is documented in `deployment-config.md`. Do not skip gates without reading why they're there.**

#### Step 2: Execute deploy_command

Run the `deploy_command` from config. Stream output. Capture exit status.

If deploy fails:

1. Read the error
2. Diagnose root cause
3. Fix the underlying issue (do NOT retry blindly)
4. Re-run from Step 0

#### Step 3: Post-Deploy Verification (THE MOST IMPORTANT STEP)

**"Deploy command exited 0" is NOT verification.** Users do not interact with your deploy command. They interact with whatever HTTP/CLI/binary surface is in front of the new revision. That is what MUST be verified.

Run ALL of these checks. Each one failing means deploy is NOT done.

1. **Revision check** — Run `deploy_check_command` again and confirm the deployed commit/revision is now `HEAD`. (Catches: deploy command succeeded but didn't actually publish the new artifact.)

2. **Traffic check** — Confirm the new revision is receiving 100% of traffic (or whatever the deploy strategy declares). For platforms with traffic splitting (Container Apps, Cloud Run, k8s): query the active traffic distribution, not just the "latest revision". (Catches: new revision exists but old revision still serves all traffic.)

3. **User-visible asset check** — Run `user_visible_check` from config. This MUST fetch the live URL with a fresh, uncached client and verify users see the new code:

   ```bash
   # Example: fetch live HTML, extract JS bundle hash, compare to expected
   LIVE_HTML=$(curl -fsSL -H "Cache-Control: no-cache" -H "Pragma: no-cache" "$LIVE_URL")
   LIVE_BUNDLE=$(echo "$LIVE_HTML" | grep -oE 'index-[A-Za-z0-9_-]+\.js' | head -1)
   EXPECTED_BUNDLE=$(cat dist/index.html | grep -oE 'index-[A-Za-z0-9_-]+\.js' | head -1)
   if [ "$LIVE_BUNDLE" != "$EXPECTED_BUNDLE" ]; then
     echo "✗ CACHE/ROUTING FAILURE: live=$LIVE_BUNDLE expected=$EXPECTED_BUNDLE"
     exit 1
   fi
   ```

   Catches: CDN cache, browser cache headers wrong, service worker stale, traffic split misconfigured, wrong revision activated.

4. **Smoke test** — Run `smoke_test_command` if declared. This is functional verification of the live endpoint, not just asset hash matching.

5. **Cache invalidation** — If the user_visible_check fails on first attempt and cache is suspected, run `cache_invalidation_command` from config (e.g., `aws cloudfront create-invalidation`, `wrangler purge`, etc.) and re-run user_visible_check. If it STILL fails, do NOT mark deploy as complete — investigate routing.

6. **Only after ALL checks pass**: write the deployed commit SHA to `deploy_state_file` from config (typically `deploy/.last-deployed`)

7. **Document**: Update `deploy/deployments/YYYY-MM-DD-HHMMSS.md` with: commit, environment, gates run, all check results (revision/traffic/user-visible/smoke), cache invalidations performed if any.

If `user_visible_check` fails after cache invalidation, deploy is NOT done. See `skills/10-deployment-git/application-deployment.md` § Cache Layers To Check for the L1-L7 troubleshooting flow.

#### Step 4: Document

Add a `DEPLOY` journal entry: what was deployed, smoke test result, any cache invalidations performed.

### Check Mode (`/deploy --check`)

Drift detection only — no deployment side effects. Useful for /wrapup, before commits, or after pulling new changes.

1. Run `deploy_check_command` to get currently-deployed commit
2. Compare to `git rev-parse HEAD`
3. Run `git diff <deployed_commit> HEAD -- <production_paths>` to summarize drift
4. Output a clear status:
   - **`✓ in sync`** — deployed commit matches HEAD
   - **`⚠ drift: N production-touching commits behind`** — list the commits, list the production files changed
   - **`✗ unknown`** — config command failed; explain why

Used by `/wrapup` to detect "committed but not deployed" before allowing session end.

## Critical Rules — see `rules/deploy-hygiene.md`

The hygiene rule loads automatically when production code is touched. Key principles:

- **"Committed" is NOT "done."** Only "live in production" is done for production-touching changes.
- **NEVER** end a session with committed-but-not-deployed production code unless the human explicitly defers
- **NEVER** skip pre-deploy gates without a documented reason
- **ALWAYS** verify deploy state BEFORE committing further production changes (don't pile on top of un-deployed code)
- **ALWAYS** run `/deploy --check` as part of the commit ritual for production-touching changes
- **ALWAYS** update `deploy/.last-deployed` on successful deploy so the check mode works

## Agent Teams

- **release-specialist** — drives onboard mode, runs execute mode, writes deployment runbook
- **security-reviewer** — pre-deploy security audit if any deploy/, secrets, or auth code changed
- **testing-specialist** — verify smoke tests run and pass post-deploy

## Skill References

- `skills/10-deployment-git/application-deployment.md` — onboarding flow + deployment-config.md schema
- `skills/10-deployment-git/deployment-cloud.md` — cloud platform patterns (Container Apps, Cloud Run, Fly, Vercel, k8s)
- `rules/deploy-hygiene.md` — the "committed ≠ deployed" rule
