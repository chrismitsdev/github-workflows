# github-workflows

Standalone GitHub Actions workflows for personal projects.

### cleanup-vercel-deployments.yml
Keeps Vercel from piling up preview deployments for Renovate branches that no longer exist.  
It runs daily at 03:00 UTC, or manually from the Actions tab with an optional `dry_run` input that only logs what it would delete.

#### How it works
1. Pages through the Vercel team's deployments (up to 20 pages of 100) and keeps only those built from branches starting with `renovate/` that have GitHub repo metadata.
2. For each unique repo and branch pair, asks the GitHub API (using a read-only token) whether the repo is readable and whether the branch still exists.
3. Acts on the result: 
   - Repo not readable: skipped.
   - Branch returns `200`: deployments are kept.
   - Branch returns `404`: every Vercel deployment tied to that branch is deleted.

Each outcome is logged as `SKIP`, `KEEP`, `DRY-RUN`, `DELETED` or `FAILED`. A failed delete doesn't stop the rest of the run.

#### Setup
| Type | Name | Purpose |
| ---- | ---- | ------- |
| Secret | `VERCEL_TOKEN` | Vercel token scoped to the team |
| Secret | `GH_READ_TOKEN` | Fine-grained PAT with `Contents: read` on every repo deployed to Vercel (expires Jan 8 2027) |
| Variable | `VERCEL_TEAM_ID` | Vercel team ID (`team_...`) |

#### Notes
- Scheduled runs delete for real. Manual runs default to `dry_run: true`.
- If `GH_READ_TOKEN` expires or loses access, repos show up as `SKIP` and nothing is deleted.
