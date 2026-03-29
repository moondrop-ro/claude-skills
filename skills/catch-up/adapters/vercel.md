# Vercel Adapter

Gather recent deployment activity from Vercel.

## Inputs

- **Repo path**: absolute path to the repository (used to find Vercel project linkage)
- **Time boundary**: ISO 8601 cutoff timestamp, or "full" for entire history

## Prerequisites

This adapter requires:
- Vercel CLI available (`vercel`)
- The repo must be linked to a Vercel project (a `.vercel/` directory should exist in the repo, or the user has run `vercel link`)

If the Vercel CLI is not available, skip this adapter and note: "Vercel adapter skipped — CLI not found."

## What to Check

### 1. Recent deployments

```bash
vercel list --cwd <repo-path> --limit 10
```

This shows recent deployments with status (Ready, Error, Building, Queued, Canceled), URL, branch, and timestamp.

Filter to deployments after the cutoff timestamp. If `--full`, show all returned.

### 2. Production deployment status

```bash
vercel inspect <production-url> --cwd <repo-path>
```

Get the production URL from `vercel list` output (the one marked as production). Check if the current production deployment is healthy.

## How to Interpret

- **Successful deploys**: note what was deployed and when. If multiple deploys happened, summarize the progression.
- **Failed deploys**: flag prominently with the error context. A failed production deploy is a high-priority finding.
- **Building/Queued deploys**: note that a deploy is in progress.
- **Preview vs Production**: distinguish between preview deployments (PRs) and production deployments (main branch).

## Output Shape

Report back to the orchestrator:

- **Deployments**: count and summary of recent deployments, noting successes and failures
- **Production status**: current state of the production deployment
- **Warnings**: failed builds, stuck deploys
- **Last successful deploy**: timestamp and what was included
