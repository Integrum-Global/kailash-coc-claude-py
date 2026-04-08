---
paths:
  - "**/Dockerfile"
  - "**/*.dockerfile"
  - "deploy/**"
  - "**/k8s/**"
  - "**/kubernetes/**"
  - "**/helm/**"
  - "**/.github/workflows/**"
  - "**/fly.toml"
  - "**/vercel.json"
  - "**/app.yaml"
  - "**/serverless.yml"
  - "**/wrangler.toml"
  - "**/Procfile"
  - "**/next.config.*"
  - "**/vite.config.*"
  - "**/package.json"
  - "deploy/deployment-config.md"
---

# Deploy Hygiene — Committed ≠ Deployed

For full DO/DO NOT examples, the 10-step checklist, the deployment-config.md schema, frontend deployment patterns (Vite, Docker, Next.js), and cache-layer troubleshooting, see `skills/10-deployment-git/application-deployment.md`. This rule loads only when infrastructure files are touched; the verbose details live in the skill.

## The Failure Mode

After every commit that touches production code, the failure pattern is identical:

1. Agent edits production file → commits → reports "done" → moves on
2. Production is still running the previous bundle — the fix never reached users
3. Next session inherits the assumption that the fix is live, builds on top of it, adds more committed-but-not-deployed code

This happens **100% of the time** without enforcement, because the agent treats `git commit` as the natural endpoint of an edit task.

## The Principle: Users See It Or It's Not Done

For any change touching production code, the definition of "done" is **users are seeing the new code from outside the system**, observable via an HTTP fetch / live URL / running binary. NOT "command exited 0".

Six levels of failure all map to "not done":

| L   | Wrong proof of "done"                              | Real proof                                                                         |
| --- | -------------------------------------------------- | ---------------------------------------------------------------------------------- |
| L1  | `git commit` returned 0                            | `/deploy` ran                                                                      |
| L2  | `kubectl apply` returned 0                         | New revision is receiving 100% of production traffic                               |
| L3  | "Container restarted" in logs                      | `curl https://prod/...` returns the new bundle hash                                |
| L4  | `vite build` returned 0 (after bypassing `tsc -b`) | The project's **declared** build command ran AND succeeded with no checks bypassed |
| L5  | Docker image built successfully                    | The Dockerfile actually rebuilt the source — NOT `COPY dist/` of stale artifacts   |
| L6  | "BUILD SUCCEEDED" reported (any number of times)   | A build was followed in the SAME response by an actual `/deploy` and verification  |

The deeper anti-pattern: when a command fails, the agent reaches for the fastest workaround that makes the immediate command return 0, then reports success based on that exit code. By the time the user looks, the original problem is buried under 4-5 workarounds and production is still broken. This is also a Zero-Tolerance Rule 1 violation (pre-existing failures MUST be fixed, not bypassed).

## MUST Rules

### 1. Verify deploy state before stacking more production commits

Run `/deploy --check` before committing additional production-touching changes. If drift is detected, do NOT pile new commits on top of un-deployed code — it makes targeted rollback impossible.

**Why:** Stacking undeployed commits creates a deployment unit larger than any individual change. If a later commit introduces a bug, you can't revert just it because production still has the pre-original state.

### 2. /deploy (or document deferral) after committing production code

When a commit touches production code, MUST run `/deploy` and verify, OR explicitly defer with a documented reason in session notes, BEFORE reporting the work as complete.

**Why:** A committed security fix that hasn't shipped is identical to no fix at all from the user's perspective.

### 3. Verify what users see, not what the deploy command returned

After `/deploy` completes, MUST verify the new code is reaching users via an external observation (curl + bundle hash compare, build-stamped health endpoint) — NOT by trusting the deploy command's exit code or container restart logs. Web frameworks: see the bundle hash extraction reference table in `skills/10-deployment-git/application-deployment.md`.

**Why:** Deploy commands return 0 in many failure modes that leave users on stale code (CDN cache, browser cache, service worker, traffic split misconfigured, wrong revision activated, wrong image tag). The only proof is fetching from outside the system.

### 4. /wrapup blocks on undeployed production code

`/wrapup` MUST run `/deploy --check`. If drift is detected, wrapup MUST require either an actual `/deploy` or documented deferral with explicit human acknowledgment.

**Why:** Without this gate, every session ends with "did the agent deploy?" unanswered. The next session inherits unverifiable state.

### 5. Pre-deploy gates run before every deploy

The gates declared in `deploy/deployment-config.md` (tests, lint, security scan, build) MUST run before each `/deploy`. NEVER `--skip-gates` without a documented reason, follow-up todo, AND human explicit acknowledgment.

**Why:** Skipping a failing test gate ships untested code. Each skip is a known unknown promoted to production.

### 6. Use the project's declared build command — never improvise

The build command is whatever `package.json` `scripts.build` (or `Cargo.toml`, or `Makefile`, etc.) declares. MUST run as-is. Substituting a "simpler" command that bypasses checks is BLOCKED.

```bash
# package.json: "build": "tsc -b && vite build"

# DO:
npm run build              # honest failure on TS errors

# DO NOT:
vite build                 # BLOCKED — bypasses tsc -b
npx vite build             # BLOCKED — same
```

**Why:** Skipping `tsc -b` ships type-broken code. This is a Zero-Tolerance Rule 1 violation: pre-existing errors must be FIXED, not bypassed.

### 7. NEVER run a build command outside `/deploy`

Build is a step inside `/deploy`. MUST NOT run `npm run build`, `docker build`, `vite build`, or any equivalent production build command as a standalone action. If a build is needed, run `/deploy`, which runs the build as its first step and continues through all verification phases.

**Why:** L6 — agent reports "BUILD SUCCEEDED" four times in a row without ever deploying because each loop iteration treats build success as a stopping point. By bundling build into `/deploy`, there is no "BUILD SUCCEEDED" stopping point, only "DEPLOY VERIFIED" or "FAILED AT STEP N".

For dev inner loop, use `npm run dev` / `cargo watch` / dev server — NEVER production build commands.

### 8. Print and follow the 10-step deploy checklist

Every `/deploy` MUST start by printing the 10-step checklist (defined in `commands/deploy.md`) and check off boxes as each step passes. The agent MUST NOT report deploy as complete until every box is checked. If any step fails, the response says "DEPLOY FAILED AT STEP N: <reason>" — NOT "build succeeded, will redeploy soon".

**Why:** Without a visible per-step checklist, the agent's mental model collapses into "the most recent command I ran". A printed checklist forces tracking and lets the user spot incomplete steps.

### 9. Dockerfile MUST rebuild from source

`COPY dist/`, `COPY build/`, `COPY out/`, `COPY .next/` in Dockerfiles is BLOCKED. The Dockerfile MUST contain the build step (`RUN npm run build` or equivalent) so deploys always rebuild from committed source. See `skills/10-deployment-git/application-deployment.md` § Frontend Deployment Patterns for the multi-stage Dockerfile patterns for Vite, Next.js (standalone/export/Vercel), and other frameworks.

**Why:** `COPY dist/` ships whatever happens to be on the developer's disk, which may be hours or days old — exact recent failure: `index-CxDD2r9Y.js` was 2 days stale, deploy "succeeded", production served the 2-day-old bundle.

### 10. Update deploy state file ONLY after user-visible check passes

Successful `/deploy` MUST update `deploy/.last-deployed` (or whatever `deploy_state_file` is declared in config) with the commit SHA — but ONLY after the user-visible check (rule 3) passes. Writing the state file based on the deploy command's exit code alone defeats drift detection.

**Why:** If the state file is updated when deploy command succeeded but users still see old code, the next `/deploy --check` will report "✓ in sync" while production is broken.

## MUST NOT

- **Report production work as "done" without deploying.** Phrases "Done — committed and pushed" / "Fix is in main" / "Should be working now" are BLOCKED. Required: "Deployed at <commit>, smoke test passed" or "Committed; deploy deferred because <documented reason>".
- **Use `--skip-gates` as a default.** Emergency hotfix only, with explicit human authorization and a same-session follow-up plan.
- **Commit production code if deploy is broken.** If `/deploy --check` returns "✗ unknown", STOP and fix the deploy mechanism first. Do not commit on top of unknown deploy state.

## Exceptions

- **SDK/library repos** (`type: sdk` in `deployment-config.md`) → use `/release` instead. This rule still applies, but "deployed" means "published to PyPI/crates.io/npm".
- **No `deployment-config.md` exists** → run `/deploy --onboard` first.
- **Legacy prose-only `deployment-config.md` (no YAML frontmatter)** → run `/deploy --onboard` to migrate; until migrated, the agent flags this in session notes and falls back to manual verification.
