# Deploy Database Backups

## Purpose

Install an automated hourly backup system that discovers all SQLite databases, encrypts them, and uploads to Google Drive. Includes restore script and failure alerts.

**When to use:** After any databases are created. Deploy early -- data loss prevention is critical.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "gog drive files list --max-results 1 2>/dev/null"
cmd --session <id> --command "find ${WORKSPACE} -name '*.db' -o -name '*.sqlite' 2>/dev/null | head -10"
cmd --session <id> --command "which gpg 2>/dev/null"
```
- Does the client have Google Drive access? If not, where should backups go?
- How many databases exist to back up?
- Is GPG available for encryption?

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Alternative | When to Adapt |
|-----------|---------|-------------|---------------|
| Backup destination | Google Drive ("OpenClaw Backups" folder) | AWS S3, Backblaze B2, local directory, rsync to remote, Dropbox | Client doesn't use Google Drive |
| Encryption | GPG symmetric encryption | age, openssl, no encryption (local only) | Different encryption tools or requirements |
| Backup frequency | Hourly | Every 6 hours, daily, on-demand only | Client's data change rate and cost sensitivity |
| Retention | Last 7 backups | More/fewer, time-based (30 days) | Client's recovery requirements |
| JSONL log backup | Included (unified stream + per-event) | Exclude logs, or separate log backup | Client wants smaller backups |
| Failure alerts | Telegram cron-updates topic | Discord, Slack, email, PagerDuty | Client's alert preference |
| Discovery paths | Auto-discover all .db/.sqlite | Explicit file list | Client wants to exclude certain databases |
| Restore workflow | Download from Drive + manifest | Restore from S3, local copy | Matches backup destination |

## Prerequisites

- `deploy-google-workspace.md` completed (Google Drive access)
- `deploy-messaging-setup.md` completed (Telegram cron-updates topic for failure alerts)
- SQLite databases exist in workspace

## What Gets Installed

### Backup Script (`scripts/backup-databases.sh`)

- Auto-discovers all .db and .sqlite files across workspace (no manual config)
- Bundles into encrypted tar archive
- Uploads to Google Drive "OpenClaw Backups" folder
- Creates `manifest.json` for restore (maps filenames to original paths)
- Keeps last 7 backups (rolling retention)
- Also backs up JSONL event logs (including unified stream all.jsonl)
- Telegram alert on any failure (cron-updates topic, ID: 1126)

Discovery paths scanned:

| Path | What's Found |
|------|-------------|
| ~/clawd/crm/data/ | CRM contacts.db |
| ~/clawd/data/ | Video pitches, knowledge base, other data stores |
| ~/.openclaw/ | Agent config databases |

### Restore Script (`scripts/restore-databases.sh`)

- Downloads from Google Drive using manifest.json
- Decrypts the tar archive
- Restores to original paths (as recorded in manifest)
- Supports `--list` (preview what would be restored without writing) and `--force` (skip confirmation prompts) modes
- Validates checksums before overwriting existing files

### Manifest Format (`manifest.json`)

```json
{
  "timestamp": "2026-02-23T10:00:00Z",
  "databases": [
    {
      "filename": "contacts.db",
      "original_path": "/home/user/clawd/crm/data/contacts.db",
      "size_bytes": 5242880,
      "checksum": "sha256:abc123..."
    }
  ],
  "logs": [
    {
      "filename": "all.jsonl",
      "original_path": "/home/user/.openclaw/logs/all.jsonl",
      "size_bytes": 1048576
    }
  ]
}
```

### Cron Jobs

| Job | Schedule |
|-----|----------|
| Hourly Database Backup | Every hour |

## Steps

### 1. Create Google Drive Backup Folder

Check if the "OpenClaw Backups" folder exists on Google Drive, create it if not.

```
cmd --session <id> --command "gog drive list --query 'name=\"OpenClaw Backups\" and mimeType=\"application/vnd.google-apps.folder\"'"
```

If the folder does not exist:

```
cmd --session <id> --command "gog drive mkdir 'OpenClaw Backups'"
```

Record the folder ID for the backup script config.

### 2. Install Backup Script

Verify the backup script is in place and executable.

```
cmd --session <id> --command "ls -la ~/clawd/scripts/backup-databases.sh"
cmd --session <id> --command "chmod +x ~/clawd/scripts/backup-databases.sh"
```

### 3. Install Restore Script

Verify the restore script is in place and executable.

```
cmd --session <id> --command "ls -la ~/clawd/scripts/restore-databases.sh"
cmd --session <id> --command "chmod +x ~/clawd/scripts/restore-databases.sh"
```

### 4. Run a Manual Backup

Execute the backup script manually to verify it works end-to-end.

```
cmd --session <id> --command "./scripts/backup-databases.sh"
```

Watch for:
- Database discovery output (should list all .db/.sqlite files found)
- Encryption step completion
- Google Drive upload confirmation
- Manifest generation

### 5. Verify Backup on Google Drive

Check that the backup archive appears in the Google Drive folder.

```
cmd --session <id> --command "gog drive list --parent 'OpenClaw Backups' --limit 5"
```

### 6. Set Up Hourly Cron Job

Schedule the backup to run every hour.

```
cmd --session <id> --command "openclaw cron add --name 'Hourly Database Backup' --schedule '0 * * * *' --command '~/clawd/scripts/backup-databases.sh'"
```

### 7. Configure Failure Alerts

Ensure the backup script sends failure notifications to the Telegram cron-updates topic (ID: 1126).

```
cmd --session <id> --command "openclaw config set backup.failure-topic 1126"
```

## Verification

Run these commands to confirm the backup system is fully operational:

```
cmd --session <id> --command "~/clawd/scripts/backup-databases.sh"
```

Check Google Drive for the new archive:

```
cmd --session <id> --command "gog drive list --parent 'OpenClaw Backups' --limit 5"
```

Verify manifest lists all databases:

```
cmd --session <id> --command "~/clawd/scripts/restore-databases.sh --list"
```

Expected output:
- All .db and .sqlite files discovered and included in backup
- Encrypted archive uploaded to Google Drive
- Manifest shows correct original paths and checksums
- Restore `--list` shows all databases with their target paths

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Drive upload fails | gog auth expired or Drive scope missing | Re-authenticate: `gog auth login`. Verify Drive access: `gog drive list --limit 1`. |
| No databases found | Discovery paths don't match actual locations | Check the discovery paths in backup-databases.sh. Add any custom database paths. |
| Encryption errors | tar or encryption tools not available | Verify tools: `which tar gpg`. Install if missing via package manager. |
| Restore fails | Manifest corrupted or Drive folder missing | Download manifest manually and inspect. Verify the "OpenClaw Backups" folder exists on Drive. |
| Backup too large | Binary blobs or large log files included | Check what's being discovered. Exclude large non-essential files in the script config. |
| Rolling retention not working | Old backups not being cleaned up | Check the retention logic in backup-databases.sh. Should keep last 7 backups. |
| Failure alert not sent | Telegram topic ID misconfigured | Verify: `openclaw config get backup.failure-topic`. Should be 1126. |

## Dependencies

- **Requires:** `deploy-google-workspace.md`, `deploy-messaging-setup.md`
- **Should deploy after:** Any database-creating system (CRM, video pitches, knowledge base, etc.)
