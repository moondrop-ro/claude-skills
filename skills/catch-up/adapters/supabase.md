# Supabase Adapter

Gather recent database migration and schema activity from Supabase.

## Inputs

- **Repo path**: absolute path to the repository
- **Time boundary**: ISO 8601 cutoff timestamp, or "full" for entire history

## Prerequisites

This adapter requires:
- Supabase CLI available (`npx supabase`)
- A `supabase/` directory in the repo (standard Supabase project structure)

If no `supabase/` directory exists, skip this adapter and note: "Supabase adapter skipped — no supabase/ directory found."

## What to Check

### 1. Migration files

```bash
ls -lt <repo-path>/supabase/migrations/
```

List migration files sorted by modification time. Filter to files modified after the cutoff. If `--full`, list all.

For each recent migration file, read its contents to understand what schema changes were made:

```bash
cat <repo-path>/supabase/migrations/<filename>.sql
```

### 2. Migration status

```bash
npx supabase migration list --workdir <repo-path>
```

This shows which migrations have been applied vs pending. Pending migrations are important to flag.

### 3. Current schema diff (if Supabase CLI is linked to a project)

```bash
npx supabase db diff --workdir <repo-path>
```

Shows differences between local schema and remote database. If there are diffs, the local and remote are out of sync.

## How to Interpret

- **New migration files**: schema is evolving. Note what tables/columns/functions were added, modified, or dropped.
- **Pending migrations**: the remote database hasn't been updated yet. Flag this — it may need to be applied before or after a deploy.
- **Schema drift**: if `db diff` shows changes, local and remote are out of sync. This could be intentional (migration not yet applied) or a problem (manual remote changes).

## Output Shape

Report back to the orchestrator:

- **Recent migrations**: summary of schema changes since the cutoff
- **Pending migrations**: migrations that haven't been applied to the remote database
- **Schema sync status**: whether local and remote are in sync
- **Warnings**: dropped tables/columns, pending migrations that might block deploys
