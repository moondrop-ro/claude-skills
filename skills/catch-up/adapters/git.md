# Git Adapter

Gather recent git activity for a repository.

## Inputs

- **Repo path**: absolute path to the git repository
- **Time boundary**: ISO 8601 cutoff timestamp, or "full" for entire history

## What to Check

Run all commands from the repo path. If a command fails, note the failure and continue with the others.

### 1. Recent commits

```bash
git -C <repo-path> log --oneline --since="<cutoff>" --all
```

If `--full`, omit the `--since` flag.

Note: for large repos with `--full`, limit to the most recent 100 commits:

```bash
git -C <repo-path> log --oneline --all -100
```

### 2. Active branches

```bash
git -C <repo-path> branch -a --sort=-committerdate
```

Report branches with recent activity (commits after the cutoff). Ignore stale branches.

### 3. Current branch status

```bash
git -C <repo-path> status --short
```

Report uncommitted changes if any.

### 4. Open pull requests (requires `gh` CLI)

```bash
gh pr list --repo <owner/repo> --state open --limit 10
```

To get the remote repo identifier:

```bash
git -C <repo-path> remote get-url origin
```

If `gh` is not available or the repo has no GitHub remote, skip this step and note: "PR check skipped — gh CLI not available or no GitHub remote."

### 5. CI status for recent PRs

```bash
gh pr checks <pr-number> --repo <owner/repo>
```

Run for each open PR from step 4. If any checks are failing, flag them.

## How to Interpret

- **Commits**: group by author if multiple contributors. Note the overall direction of work (what areas of the codebase are being changed).
- **Branches**: active feature branches suggest in-flight work. Branches with no recent commits may be stale.
- **Uncommitted changes**: the repo has work-in-progress that hasn't been saved yet.
- **Open PRs**: work waiting for review or merge. Check if any are stale (no activity in 3+ days).
- **CI failures**: flag prominently — these may block work.

## Output Shape

Report back to the orchestrator:

- **Commits**: count and summary of recent commits, grouped by author
- **In-flight work**: active branches and open PRs
- **Warnings**: CI failures, stale PRs, uncommitted changes
- **Overall direction**: 1-2 sentence summary of what the codebase is moving toward
