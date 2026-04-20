# Deploy Git Auto-Sync

## Purpose

Install hourly automatic git commit and push for workspace changes. Includes conflict detection with notification, pre-commit hook for sensitive data prevention, and dual-repo sync.

**When to use:** After initial workspace setup. Prevents work loss and provides change history.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "git -C ${WORKSPACE} remote -v 2>/dev/null"
cmd --session <id> --command "git -C ${WORKSPACE} status --short 2>/dev/null"
cmd --session <id> --command "ls ${WORKSPACE}/.git/hooks/pre-commit 2>/dev/null"
```
- Is git configured with a remote?
- Are there existing uncommitted changes?
- Is there already a pre-commit hook? (Don't overwrite without checking)

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Alternative | When to Adapt |
|-----------|---------|-------------|---------------|
| Git host | GitHub (SSH or HTTPS) | GitLab, Bitbucket, self-hosted | Client uses a different git host |
| Repos synced | Two repos: `~/.openclaw/` + `${WORKSPACE}` | Single repo, or different paths | Client's repo structure |
| Sync frequency | Hourly | Every 6 hours, daily, manual only | Client preference or commit noise concern |
| Conflict handling | Detect and notify (no force resolution) | Auto-merge, or pause sync | Client's preference for conflict handling |
| Pre-commit hook | Block .env, Chrome profiles, large binaries | Custom blocklist, or no hook | Client's sensitive file patterns |
| Failure alerts | Telegram | Discord, Slack | Client's messaging platform |
| Branch | Main branch | Different branch per client | Multi-environment setup |

## Prerequisites

- Git configured on the client (`user.name`, `user.email` set)
- GitHub SSH key or HTTPS credentials configured
- Remote repository set up for both repos
- `deploy-messaging-setup.md` completed (Telegram for failure notifications)

## What Gets Installed

### Auto-Sync Script (`scripts/auto-git-sync.sh`)

- Commits all workspace changes with timestamped message
- Syncs two repos in order: `~/.openclaw/` (config), `~/clawd/` (workspace)
- Pulls before pushing (detects and handles conflicts)
- Conflict detection with Telegram notification (no force resolution -- requires manual intervention)
- Validates main branch and remote state before push
- Uses cron-log for tracking
- Uses workspace-state.js for PID-based locking (prevents concurrent sync runs)

Sync order:

| Step | Action |
|------|--------|
| 1 | Acquire PID lock via workspace-state.js |
| 2 | Sync `~/.openclaw/` (config repo): stage, commit, pull, push |
| 3 | Sync `~/clawd/` (workspace repo): stage, commit, pull, push |
| 4 | Release PID lock |
| 5 | Log result to cron-log |

### Pre-Commit Hook (`scripts/pre-commit`)

Prevents committing sensitive or problematic files.

| Check | What's Blocked |
|-------|---------------|
| .env files | Any file matching `*.env` or `.env*` patterns |
| Chrome profile data | Cookies, session tokens, login data from browser profiles |
| Large binaries | Files over 10MB (configurable threshold) |
| Credentials | Files matching `*credentials*`, `*secret*`, `*token*` patterns |

The hook prints a clear message explaining what was blocked and why.

### Cron Jobs

| Job | Schedule |
|-----|----------|
| Auto Git Sync | Hourly |

## Steps

### 1. Verify Git Remote Is Configured

Check that both repos have remotes set up.

```
cmd --session <id> --command "cd ~/.openclaw && git remote -v"
cmd --session <id> --command "cd ~/clawd && git remote -v"
```

Both should show a `push` remote. If not, configure them:

```
cmd --session <id> --command "cd ~/.openclaw && git remote add origin <remote-url>"
cmd --session <id> --command "cd ~/clawd && git remote add origin <remote-url>"
```

### 2. Verify Git Identity

```
cmd --session <id> --command "git config --global user.name && git config --global user.email"
```

### 3. Install Auto-Sync Script

Verify the script is in place and executable.

```
cmd --session <id> --command "ls -la ~/clawd/scripts/auto-git-sync.sh"
cmd --session <id> --command "chmod +x ~/clawd/scripts/auto-git-sync.sh"
```

### 4. Install Pre-Commit Hook

Install the hook in both repos.

```
cmd --session <id> --command "cp ~/clawd/scripts/pre-commit ~/.openclaw/.git/hooks/pre-commit && chmod +x ~/.openclaw/.git/hooks/pre-commit"
cmd --session <id> --command "cp ~/clawd/scripts/pre-commit ~/clawd/.git/hooks/pre-commit && chmod +x ~/clawd/.git/hooks/pre-commit"
```

Test the hook:

```
cmd --session <id> --command "cd ~/clawd && echo 'test' > .env.test && git add .env.test && git commit -m 'test hook' 2>&1; rm .env.test; git reset HEAD .env.test 2>/dev/null"
```

The commit should be blocked with a clear message about .env files.

### 5. Run a Manual Sync

Execute the sync script manually to verify it works.

```
cmd --session <id> --command "~/clawd/scripts/auto-git-sync.sh"
```

Check the output for:
- Both repos staged and committed (or "nothing to commit")
- Pull completed without conflicts
- Push completed successfully

### 6. Set Up Hourly Cron Job

```
cmd --session <id> --command "openclaw cron add --name 'Auto Git Sync' --schedule '0 * * * *' --command '~/clawd/scripts/auto-git-sync.sh'"
```

### 7. Configure Telegram Failure Notification

Ensure sync failures send alerts to the cron-updates topic (ID: 1126).

```
cmd --session <id> --command "openclaw config set git-sync.failure-topic 1126"
```

## Verification

Run these commands to confirm the auto-sync system is operational:

```
cmd --session <id> --command "~/clawd/scripts/auto-git-sync.sh"
cmd --session <id> --command "cd ~/clawd && git log --oneline -5"
cmd --session <id> --command "cd ~/.openclaw && git log --oneline -5"
```

Expected output:
- Sync completes without errors
- Recent commits show timestamped auto-sync messages
- Remote repos are up to date

Verify pre-commit hook is installed:

```
cmd --session <id> --command "ls -la ~/clawd/.git/hooks/pre-commit ~/.openclaw/.git/hooks/pre-commit"
```

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Push rejected | Remote has changes not in local | The script pulls before pushing. If still rejected, check for force-push protection on the remote. Manual pull and merge may be needed. |
| Pre-commit blocks commit | Sensitive file detected | This is intentional. Review what's being committed. Add the file to .gitignore if it should never be committed. |
| Conflict notification | Two machines edited the same file | Manual intervention required. SSH into the machine, resolve the conflict, then commit. Do not force push. |
| Stale PID lock | Previous sync crashed without releasing lock | Check workspace-state.js for the PID. If the process is dead, clear the lock: `node ~/clawd/scripts/workspace-state.js --clear-lock`. |
| Auth failure on push | SSH key or HTTPS token expired | Re-configure credentials. For SSH: check `ssh -T git@github.com`. For HTTPS: update the credential helper. |
| Only one repo syncing | Script exiting early after first repo | Check the script for error handling. First repo failure should not prevent second repo sync. |

## Dependencies

- **Requires:** Git configured, `deploy-messaging-setup.md`
