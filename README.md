# github-workflows

Standalone GitHub Actions workflows for personal projects.

## cleanup-vercel-deployment.yaml
Keeps Vercel from piling up preview deployments for Renovate branches that no longer exist.  
It runs twice daily at 03:23 and 15:23 UTC (GitHub scheduled runs can be delayed or dropped, so the second run is a safety net), or manually from the Actions tab with an optional `dry_run` input that only logs what it would delete. 

### How it works
1. Pages through the Vercel team's deployments (up to 20 pages of 100) and keeps only non-production deployments built from branches starting with `renovate/` that have GitHub repo metadata. A `WARN` line is logged if the page cap is hit and older deployments went unchecked.
2. For each unique repo and branch pair, asks the GitHub API (using a read-only token) whether the repo is readable and whether the branch still exists.
3. Acts on the result:
   - Repo not readable: skipped.
   - Branch returns `200`: deployments are kept.
   - Branch returns `404`: every Vercel deployment tied to that branch is deleted.

Each outcome is logged as `SKIP`, `KEEP`, `DRY-RUN`, `DELETED` or `FAILED`.  
A failed delete doesn't stop the rest of the run, but the job exits with an error so the failure is visible.

### Setup
| Type     | Name             | Purpose |
| -------- | ---------------- | ------- |
| Secret   | `VERCEL_TOKEN`   | Vercel token scoped to the team |
| Secret   | `GH_READ_TOKEN`  | Fine-grained PAT with `Contents: read` on every repo deployed to Vercel (expires Jan 8 2027) |
| Variable | `VERCEL_TEAM_ID` | Vercel team ID (`team_...`) |

### Notes
- Scheduled runs delete for real. Manual runs default to `dry_run: true`.
- The run aborts immediately if `VERCEL_TOKEN` or `VERCEL_TEAM_ID` is missing.
- If `GH_READ_TOKEN` expires or loses access, repos show up as `SKIP` and nothing is deleted.

## prune-vercel-deployments.yaml
Keeps only the most recent Vercel deployments: the last 3 production deployments per project and the last preview deployment per branch.  
It runs twice daily at 03:41 and 15:41 UTC, or manually from the Actions tab with an optional `dry_run` input (default `true`).  
It shares the `vercel-cleanup` concurrency group with `cleanup-vercel-deployment.yaml`, so the two never delete at the same time.

### How it works
1. Pages through the Vercel team's `READY` deployments, newest first (up to 50 pages of 100). A `WARN` line is logged if the page cap is hit; older deployments are then left alone.
2. Groups them per project: production in one group, previews per branch (or per project if `PREVIEW_SCOPE: project`). Renovate previews and custom environments are skipped, since `cleanup-vercel-deployment.yaml` handles Renovate.
3. Deletes everything past the newest `KEEP_PRODUCTION` (3) / `KEEP_PREVIEW` (1) in each group.
4. Never deletes the deployment currently serving production, even after an Instant Rollback. If it can't be determined, production deployments for that project are skipped.

Each outcome is logged as `SKIP`, `KEEP`, `DRY-RUN`, `DELETED` or `FAILED`.

### Setup
Uses the same `VERCEL_TOKEN` secret and `VERCEL_TEAM_ID` variable as above. No GitHub token needed.

### Notes
- Errored, canceled and building deployments are not touched; Vercel's retention policy handles those.
- Scheduled runs delete for real. Manual runs default to `dry_run: true`.
