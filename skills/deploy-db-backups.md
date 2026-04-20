# Deploy Database Backups

## Compatibility
- **OpenClaw Version**: 2026.4.15+ (bumped 2026-04-20)
- **Status**: NEEDS AUDIT
- **Blocked Commands**: `openclaw telegram send`
- **Notes**: References `openclaw telegram send` for failure alerts, which does not exist. The core backup logic (GPG encryption, gog Drive upload, cron scheduling) may work if `openclaw telegram send` is replaced with raw Telegram Bot API calls and scheduling uses platform-native methods.

## Purpose

Install an automated hourly backup system that discovers all SQLite databases in the workspace, encrypts them with GPG, and uploads to Google Drive. Includes a restore script and failure alerts via Telegram. Data loss prevention is critical -- deploy this early after any databases are created.

**When to use:** After any database-creating skill has been deployed (CRM, knowledge base, social tracking, etc.).

**What this skill does:**
1. Creates a Google Drive folder for backup storage
2. Writes a backup script that auto-discovers, encrypts, and uploads all SQLite databases and JSONL logs
3. Writes a restore script that downloads, decrypts, and restores from a manifest
4. Schedules hourly cron execution
5. Configures failure alerts to Telegram

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

Resolve these before executing. Sources are listed for each.

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${DRIVE_BACKUP_FOLDER_ID}` | Created in Phase 1, Step 1.1 (Google Drive folder ID) | `1aBcDeFgHiJkLmNoPqRsT` |
| `${GPG_PASSPHRASE}` | Client profile or generated during deployment `[HUMAN_INPUT]` | `strong-random-passphrase-here` |
| `${CRON_UPDATES_TOPIC_ID}` | Client profile: messaging config (Telegram cron-updates topic) | `1126` |
| `${TELEGRAM_GROUP}` | Client profile: messaging config | `-100xxxxxxxxxx` |
| `${BACKUP_RETENTION_COUNT}` | Client preference (default: 7) | `7` |
| `${AGENT_USER}` | Pre-flight check: `whoami` on client machine | `john` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**

Verify Google Drive access is working:

**Remote:**
```
gog drive files list --max-results 1 2>/dev/null && echo 'DRIVE_OK' || echo 'DRIVE_FAIL'
```

Expected: `DRIVE_OK`. If `DRIVE_FAIL`, deploy `deploy-google-workspace.md` first.

Discover existing databases to understand what will be backed up:

**Remote:**
```
find ${WORKSPACE} -name '*.db' -o -name '*.sqlite' 2>/dev/null | head -20
```

Expected: List of database files. If empty, there is nothing to back up yet -- consider deploying a data system first.

Check if GPG is available for encryption:

**Remote:**
```
which gpg 2>/dev/null && gpg --version | head -1 || echo 'NO_GPG'
```

Expected: GPG version output. If `NO_GPG`, install it (e.g., `brew install gnupg` on macOS, `sudo apt install gnupg` on Linux).

Check if backup system is already deployed:

**Remote:**
```
test -f ${WORKSPACE}/scripts/backup-databases.sh && echo 'BACKUP_SCRIPT_EXISTS' || echo 'NOT_DEPLOYED'
```

Expected: `NOT_DEPLOYED` for first-time setup. If `BACKUP_SCRIPT_EXISTS`, check if it needs updating rather than fresh install.

**Decision points from pre-flight:**
- Does the client have Google Drive access? If not, adapt the backup destination.
- How many databases exist? This affects backup size and duration.
- Is GPG available? If not, install it or use an alternative encryption tool.
- Is there an existing backup setup? If so, review and update rather than overwrite.

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Backup destination | Google Drive ("OpenClaw Backups" folder) | Client doesn't use Google Drive -- use S3, Backblaze B2, local dir, rsync, or Dropbox |
| Encryption | GPG symmetric encryption | Client prefers `age`, `openssl`, or no encryption (local-only backups) |
| Backup frequency | Hourly (`0 * * * *`) | Client's data changes slowly (daily) or quickly (every 15 min) |
| Retention count | Last 7 backups | Client needs longer history (30) or has storage constraints (3) |
| JSONL log backup | Included (unified stream + per-event logs) | Client wants smaller backups or logs are handled separately |
| Failure alerts | Telegram cron-updates topic | Client uses Discord, Slack, email, or PagerDuty instead |
| Discovery paths | Auto-discover all `.db`/`.sqlite` under `${WORKSPACE}` and `~/.openclaw/` | Client wants to exclude certain databases or include paths outside workspace |
| Restore workflow | Download from Drive + manifest | Must match backup destination (S3, local, etc.) |

## Prerequisites

- OpenClaw installed and running (`openclaw/install.md`)
- `deploy-google-workspace.md` completed (Google Drive access via `gog`)
- `deploy-messaging-setup.md` completed (Telegram cron-updates topic for failure alerts)
- GPG installed on client machine
- At least one SQLite database exists in the workspace

## What Gets Installed

| Component | Location | Purpose |
|-----------|----------|---------|
| Backup script | `${WORKSPACE}/scripts/backup-databases.sh` | Auto-discover, encrypt, upload databases |
| Restore script | `${WORKSPACE}/scripts/restore-databases.sh` | Download, decrypt, restore from manifest |
| Backup passphrase | `${WORKSPACE}/.backup-passphrase` (mode 600) | GPG symmetric encryption key |
| Cron job | `Hourly Database Backup` | Runs backup script every hour |

## Steps

### Phase 1: Prepare Backup Infrastructure

#### 1.1 Create Google Drive Backup Folder `[AUTO]`

Check if the "OpenClaw Backups" folder already exists on Google Drive, and create it if not.

**Remote:**
```
gog drive list --query 'name="OpenClaw Backups" and mimeType="application/vnd.google-apps.folder"'
```

Expected: Either an existing folder (with its ID) or empty results.

If the folder does not exist, create it:

**Remote:**
```
gog drive mkdir 'OpenClaw Backups'
```

Expected: Folder created, with its ID in the output. Record this as `${DRIVE_BACKUP_FOLDER_ID}`.

If this fails: Verify `gog` authentication with `gog auth status`. Re-authenticate with `gog auth login` if expired.

If already exists: Use the existing folder's ID. Skip creation.

#### 1.2 Set Encryption Passphrase `[HUMAN_INPUT]`

The backup archive is encrypted with GPG symmetric encryption. A passphrase is needed.

Ask the operator: "What passphrase should be used for backup encryption? Generate a strong random one if the client doesn't have a preference."

Store the passphrase in a protected file so the cron job can use it non-interactively:

**Remote:**
```
printf '%s' "${GPG_PASSPHRASE}" > ${WORKSPACE}/.backup-passphrase
chmod 600 ${WORKSPACE}/.backup-passphrase
```

Expected: File exists at `${WORKSPACE}/.backup-passphrase` with mode `600` (owner read/write only).

If this fails: Check that `${WORKSPACE}` directory exists. Create it with `mkdir -p ${WORKSPACE}`.

If already exists: Verify the existing passphrase still works by testing a round-trip encrypt/decrypt. If the client wants a new passphrase, update the file and note that old backups will require the old passphrase to restore.

### Phase 2: Create Backup Script

#### 2.1 Write the Backup Script `[GUIDED]`

Write the backup script that performs auto-discovery, encryption, and upload. This script does not exist by default -- the operator must create it as part of this deployment.

The script must implement the following logic:

1. **Discovery:** Find all `.db` and `.sqlite` files under `${WORKSPACE}` and `~/.openclaw/`
2. **JSONL logs:** Also include `~/.openclaw/logs/*.jsonl` if they exist
3. **Manifest:** Generate a `manifest.json` recording each file's original path, size, and SHA-256 checksum
4. **Archive:** Create a tar archive containing all discovered files plus the manifest
5. **Encrypt:** GPG symmetric encrypt the archive using the passphrase from `${WORKSPACE}/.backup-passphrase`
6. **Upload:** Upload the encrypted archive to the "OpenClaw Backups" folder on Google Drive using `gog drive upload`
7. **Retention:** After successful upload, list backups in the Drive folder and delete any beyond the retention count (oldest first)
8. **Cleanup:** Remove local temp files (tar, encrypted archive)
9. **Error handling:** On any failure, send an alert to Telegram and exit non-zero

**Remote:**
```
cat > ${WORKSPACE}/scripts/backup-databases.sh << 'SCRIPT_EOF'
#!/bin/bash
set -euo pipefail

# Configuration
WORKSPACE_DIR="WORKSPACE_PLACEHOLDER"
PASSPHRASE_FILE="${WORKSPACE_DIR}/.backup-passphrase"
DRIVE_FOLDER="OpenClaw Backups"
RETENTION=RETENTION_PLACEHOLDER
TELEGRAM_GROUP="TELEGRAM_GROUP_PLACEHOLDER"
TELEGRAM_TOPIC="TELEGRAM_TOPIC_PLACEHOLDER"
TIMESTAMP=$(date -u +%Y%m%d-%H%M%S)
BACKUP_NAME="openclaw-backup-${TIMESTAMP}"
TMPDIR=$(mktemp -d)

cleanup() {
  rm -rf "${TMPDIR}"
}
trap cleanup EXIT

alert_failure() {
  local msg="Backup FAILED at $(date -u +%H:%M): $1"
  # Preferred: route via openclaw channels (2026.4.15+). Falls back to raw Telegram Bot
  # API if `channels` is not configured on this host (requires TELEGRAM_BOT_TOKEN env var
  # or a file at ${WORKSPACE_DIR}/.telegram-bot-token).
  if command -v openclaw >/dev/null 2>&1 && openclaw channels list 2>/dev/null | grep -q telegram; then
    openclaw channels send --channel telegram \
      --to "${TELEGRAM_GROUP}:${TELEGRAM_TOPIC}" \
      --message "${msg}" 2>/dev/null || true
  else
    local token="${TELEGRAM_BOT_TOKEN:-}"
    [ -z "${token}" ] && [ -f "${WORKSPACE_DIR}/.telegram-bot-token" ] && \
      token="$(cat "${WORKSPACE_DIR}/.telegram-bot-token")"
    if [ -n "${token}" ]; then
      curl -sS -X POST "https://api.telegram.org/bot${token}/sendMessage" \
        --data-urlencode "chat_id=${TELEGRAM_GROUP}" \
        --data-urlencode "message_thread_id=${TELEGRAM_TOPIC}" \
        --data-urlencode "text=${msg}" >/dev/null 2>&1 || true
    fi
  fi
}

# Discovery
echo "Discovering databases..."
DATABASES=()
while IFS= read -r db; do
  [ -n "${db}" ] && DATABASES+=("${db}")
done < <(find "${WORKSPACE_DIR}" "$HOME/.openclaw" -type f \( -name '*.db' -o -name '*.sqlite' \) 2>/dev/null)

if [ ${#DATABASES[@]} -eq 0 ]; then
  echo "No databases found. Nothing to back up."
  exit 0
fi

echo "Found ${#DATABASES[@]} database(s)."

# Discover JSONL logs
LOGS=()
if [ -d "$HOME/.openclaw/logs" ]; then
  while IFS= read -r log; do
    [ -n "${log}" ] && LOGS+=("${log}")
  done < <(find "$HOME/.openclaw/logs" -type f -name '*.jsonl' 2>/dev/null)
fi

# Build manifest
echo "Building manifest..."
MANIFEST="${TMPDIR}/manifest.json"
printf '{\n  "timestamp": "%s",\n  "databases": [\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)" > "${MANIFEST}"

first=true
for db in "${DATABASES[@]}"; do
  size=$(wc -c < "${db}" | tr -d ' ')
  checksum=$(shasum -a 256 "${db}" | cut -d' ' -f1)
  basename=$(basename "${db}")
  if [ "${first}" = true ]; then first=false; else printf ',\n' >> "${MANIFEST}"; fi
  printf '    {"filename": "%s", "original_path": "%s", "size_bytes": %s, "checksum": "sha256:%s"}' \
    "${basename}" "${db}" "${size}" "${checksum}" >> "${MANIFEST}"
done

printf '\n  ],\n  "logs": [\n' >> "${MANIFEST}"

first=true
for log in "${LOGS[@]}"; do
  size=$(wc -c < "${log}" | tr -d ' ')
  basename=$(basename "${log}")
  if [ "${first}" = true ]; then first=false; else printf ',\n' >> "${MANIFEST}"; fi
  printf '    {"filename": "%s", "original_path": "%s", "size_bytes": %s}' \
    "${basename}" "${log}" "${size}" >> "${MANIFEST}"
done

printf '\n  ]\n}\n' >> "${MANIFEST}"

# Create tar archive.
# NOTE: We deliberately do NOT use `tar -C /` with manifest-derived paths. Tarring
# from filesystem root trips SELinux/Gatekeeper/AppArmor policies on hardened hosts
# and silently omits files when paths resolve through symlinks. Stage explicit files
# into a scratch directory and tar from there. This also preserves stable relative
# paths inside the archive (restore consults the manifest for original paths).
echo "Creating archive..."
TAR_FILE="${TMPDIR}/${BACKUP_NAME}.tar.gz"
STAGING="${TMPDIR}/staging"
mkdir -p "${STAGING}"
cp "${MANIFEST}" "${STAGING}/"
for db in "${DATABASES[@]}"; do
  cp "${db}" "${STAGING}/" || { alert_failure "Failed to stage ${db}"; exit 1; }
done
for log in "${LOGS[@]}"; do
  cp "${log}" "${STAGING}/" || true   # logs are best-effort
done
tar -czf "${TAR_FILE}" -C "${STAGING}" . || {
  alert_failure "tar archive creation failed"
  exit 1
}

# Encrypt
echo "Encrypting..."
ENC_FILE="${TAR_FILE}.gpg"
gpg --batch --yes --symmetric --cipher-algo AES256 \
  --passphrase-file "${PASSPHRASE_FILE}" \
  --output "${ENC_FILE}" "${TAR_FILE}" || {
  alert_failure "GPG encryption failed"
  exit 1
}

# Upload
echo "Uploading to Google Drive..."
gog drive upload "${ENC_FILE}" --parent "${DRIVE_FOLDER}" || {
  alert_failure "Drive upload failed"
  exit 1
}

# Also upload manifest (unencrypted, for quick restore listing)
cp "${MANIFEST}" "${TMPDIR}/${BACKUP_NAME}-manifest.json"
gog drive upload "${TMPDIR}/${BACKUP_NAME}-manifest.json" --parent "${DRIVE_FOLDER}" || true

# Retention: delete old backups beyond retention count.
# Use STRICT globs per file type so stray files in the folder (manual renames, partial
# uploads, half-deleted artifacts) can't shift the offset and cause wrong deletions.
# Each category is pruned independently, ordered by createdTime (mtime on Drive).
echo "Enforcing retention (keep last ${RETENTION})..."
prune_category() {
  local pattern="$1"
  local listing
  listing=$(gog drive list --parent "${DRIVE_FOLDER}" \
    --query "name contains '${pattern}'" \
    --order-by 'createdTime desc' 2>/dev/null)
  local count
  count=$(printf '%s\n' "${listing}" | grep -c "${pattern}" || true)
  if [ "${count}" -le "${RETENTION}" ]; then
    return 0
  fi
  printf '%s\n' "${listing}" | grep "${pattern}" | tail -n +$((RETENTION + 1)) | \
    while IFS= read -r line; do
      local file_id
      file_id=$(printf '%s\n' "${line}" | awk '{print $1}')
      [ -n "${file_id}" ] && gog drive delete "${file_id}" 2>/dev/null || true
    done
}
prune_category '.tar.gz.gpg'
prune_category '-manifest.json'

echo "Backup complete: ${BACKUP_NAME}"
SCRIPT_EOF
```

The quoted heredoc (`<< 'SCRIPT_EOF'`) preserves placeholders literally. Immediately substitute them with the resolved variables in a single `sed -i` pass — do not hand-edit. This follows the authoring-guide anti-pattern #11 (write-then-substitute, two commands, never expect the operator to edit a heredoc before sending):

**Remote:**
```
sed -i.bak \
  -e "s|WORKSPACE_PLACEHOLDER|${WORKSPACE}|g" \
  -e "s|RETENTION_PLACEHOLDER|${BACKUP_RETENTION_COUNT}|g" \
  -e "s|TELEGRAM_GROUP_PLACEHOLDER|${TELEGRAM_GROUP}|g" \
  -e "s|TELEGRAM_TOPIC_PLACEHOLDER|${CRON_UPDATES_TOPIC_ID}|g" \
  ${WORKSPACE}/scripts/backup-databases.sh && rm -f ${WORKSPACE}/scripts/backup-databases.sh.bak
```

Expected: The four `*_PLACEHOLDER` strings no longer appear in the file. Verify with `grep PLACEHOLDER ${WORKSPACE}/scripts/backup-databases.sh` (should be empty).

Then make it executable:

**Remote:**
```
chmod +x ${WORKSPACE}/scripts/backup-databases.sh
```

Expected: Script exists at `${WORKSPACE}/scripts/backup-databases.sh` and is executable.

If this fails: Check that `${WORKSPACE}/scripts/` directory exists. Create it with `mkdir -p ${WORKSPACE}/scripts`.

If already exists: Compare the existing script to the new version. If the logic differs, back up the existing script as `backup-databases.sh.bak` before overwriting.

#### 2.2 Write the Restore Script `[GUIDED]`

Write the restore script that downloads and decrypts from Google Drive. This script does not exist by default -- the operator must create it as part of this deployment.

The script must implement the following logic:

1. **List mode (`--list`):** Download the latest manifest from Drive and display what would be restored, without writing anything
2. **Download:** Download the latest encrypted archive from the "OpenClaw Backups" folder on Drive
3. **Decrypt:** GPG decrypt using the passphrase from `${WORKSPACE}/.backup-passphrase`
4. **Validate:** Check SHA-256 checksums from the manifest against the archive contents
5. **Confirm:** Unless `--force` is passed, show the list of files and paths and ask for confirmation
6. **Restore:** Extract files to their original paths (as recorded in the manifest)

**Remote:**
```
cat > ${WORKSPACE}/scripts/restore-databases.sh << 'SCRIPT_EOF'
#!/bin/bash
set -euo pipefail

WORKSPACE_DIR="WORKSPACE_PLACEHOLDER"
PASSPHRASE_FILE="${WORKSPACE_DIR}/.backup-passphrase"
DRIVE_FOLDER="OpenClaw Backups"
TMPDIR=$(mktemp -d)
LIST_ONLY=false
FORCE=false

for arg in "$@"; do
  case "${arg}" in
    --list) LIST_ONLY=true ;;
    --force) FORCE=true ;;
  esac
done

cleanup() {
  rm -rf "${TMPDIR}"
}
trap cleanup EXIT

# Find latest manifest on Drive
echo "Finding latest backup..."
LATEST_MANIFEST=$(gog drive list --parent "${DRIVE_FOLDER}" \
  --query 'name contains "-manifest.json"' \
  --order-by 'createdTime desc' --limit 1 2>/dev/null | head -1)

if [ -z "${LATEST_MANIFEST}" ]; then
  echo "ERROR: No backup manifests found on Drive."
  exit 1
fi

MANIFEST_ID=$(echo "${LATEST_MANIFEST}" | awk '{print $1}')
MANIFEST_NAME=$(echo "${LATEST_MANIFEST}" | awk '{print $2}')
BACKUP_NAME="${MANIFEST_NAME%-manifest.json}"

echo "Latest backup: ${BACKUP_NAME}"

# Download manifest
gog drive download "${MANIFEST_ID}" --output "${TMPDIR}/manifest.json"

echo ""
echo "=== Backup Contents ==="
python3 -c "
import json
with open('${TMPDIR}/manifest.json') as f:
    m = json.load(f)
print(f'Timestamp: {m[\"timestamp\"]}')
print(f'Databases ({len(m[\"databases\"])}):')
for db in m['databases']:
    print(f'  {db[\"filename\"]} -> {db[\"original_path\"]} ({db[\"size_bytes\"]} bytes)')
if m.get('logs'):
    print(f'Logs ({len(m[\"logs\"])}):')
    for log in m['logs']:
        print(f'  {log[\"filename\"]} -> {log[\"original_path\"]} ({log[\"size_bytes\"]} bytes)')
"

if [ "${LIST_ONLY}" = true ]; then
  echo ""
  echo "List mode -- no files were modified."
  exit 0
fi

if [ "${FORCE}" != true ]; then
  echo ""
  read -p "Restore these files to their original paths? [y/N] " confirm
  if [ "${confirm}" != "y" ] && [ "${confirm}" != "Y" ]; then
    echo "Aborted."
    exit 0
  fi
fi

# Download and decrypt archive
ARCHIVE_FILE=$(gog drive list --parent "${DRIVE_FOLDER}" \
  --query "name=\"${BACKUP_NAME}.tar.gz.gpg\"" --limit 1 | head -1)
ARCHIVE_ID=$(echo "${ARCHIVE_FILE}" | awk '{print $1}')

echo "Downloading encrypted archive..."
gog drive download "${ARCHIVE_ID}" --output "${TMPDIR}/${BACKUP_NAME}.tar.gz.gpg"

echo "Decrypting..."
gpg --batch --yes --decrypt \
  --passphrase-file "${PASSPHRASE_FILE}" \
  --output "${TMPDIR}/${BACKUP_NAME}.tar.gz" \
  "${TMPDIR}/${BACKUP_NAME}.tar.gz.gpg"

echo "Extracting..."
tar -xzf "${TMPDIR}/${BACKUP_NAME}.tar.gz" -C /

echo "Validating checksums..."
python3 -c "
import json, hashlib, sys
with open('${TMPDIR}/manifest.json') as f:
    m = json.load(f)
errors = 0
for db in m['databases']:
    path = db['original_path']
    expected = db['checksum'].replace('sha256:', '')
    try:
        actual = hashlib.sha256(open(path, 'rb').read()).hexdigest()
        if actual != expected:
            print(f'CHECKSUM MISMATCH: {path}')
            errors += 1
        else:
            print(f'OK: {path}')
    except FileNotFoundError:
        print(f'MISSING: {path}')
        errors += 1
sys.exit(1 if errors > 0 else 0)
"

echo "Restore complete."
SCRIPT_EOF
```

Substitute the workspace placeholder with the resolved variable in a single `sed -i` pass (anti-pattern #11: never hand-edit a heredoc before sending):

**Remote:**
```
sed -i.bak -e "s|WORKSPACE_PLACEHOLDER|${WORKSPACE}|g" \
  ${WORKSPACE}/scripts/restore-databases.sh && rm -f ${WORKSPACE}/scripts/restore-databases.sh.bak
```

Expected: `WORKSPACE_PLACEHOLDER` no longer appears in the file. Verify with `grep PLACEHOLDER ${WORKSPACE}/scripts/restore-databases.sh` (should be empty).

Then make it executable:

**Remote:**
```
chmod +x ${WORKSPACE}/scripts/restore-databases.sh
```

Expected: Script exists at `${WORKSPACE}/scripts/restore-databases.sh` and is executable.

If this fails: Same as backup script -- check that `${WORKSPACE}/scripts/` directory exists.

If already exists: Compare and back up existing version before overwriting.

### Phase 3: Test Backup and Restore

#### 3.1 Run a Manual Backup `[GUIDED]`

Execute the backup script manually to verify the full pipeline works end-to-end.

**Remote:**
```
${WORKSPACE}/scripts/backup-databases.sh
```

Expected: Output showing:
- Database discovery (list of `.db`/`.sqlite` files found)
- Archive creation
- GPG encryption completed
- Google Drive upload succeeded
- Retention enforcement
- "Backup complete" message

If this fails:
- **"No databases found"**: Verify discovery paths match where databases actually live. Check `find ${WORKSPACE} -name '*.db' -o -name '*.sqlite'`.
- **"GPG encryption failed"**: Check passphrase file exists and is readable: `test -f ${WORKSPACE}/.backup-passphrase && echo OK`.
- **"Drive upload failed"**: Re-check `gog` authentication: `gog auth status`. Re-authenticate if needed.
- **Permission denied**: Check script is executable: `ls -la ${WORKSPACE}/scripts/backup-databases.sh`.

#### 3.2 Verify Backup on Google Drive `[AUTO]`

Confirm the encrypted archive and manifest appeared on Drive.

**Remote:**
```
gog drive list --parent 'OpenClaw Backups' --limit 5
```

Expected: At least two files visible -- the `.tar.gz.gpg` encrypted archive and the `-manifest.json` file, both with recent timestamps.

If this fails: Check that the folder name is exactly "OpenClaw Backups" (case-sensitive). List root Drive contents to find it: `gog drive list --limit 20`.

#### 3.3 Test Restore List Mode `[AUTO]`

Verify the restore script can read the manifest and list contents without modifying anything.

**Remote:**
```
${WORKSPACE}/scripts/restore-databases.sh --list
```

Expected: Output showing the backup timestamp, list of databases with their original paths and sizes, and "List mode -- no files were modified."

If this fails: Check that `gog drive list` works and that the manifest was uploaded in Step 3.1.

### Phase 4: Schedule and Alert

#### 4.1 Schedule Hourly Cron Job `[AUTO]`

Schedule the backup script to run every hour. Check for existing cron entries first to avoid duplicates.

Check for existing backup cron entry:

**Remote:**
```
crontab -l 2>/dev/null | grep 'backup-databases' || echo 'NO_EXISTING_CRON'
```

If no existing entry, add the cron job:

**Remote:**
```
(crontab -l 2>/dev/null; echo "0 * * * * ${WORKSPACE}/scripts/backup-databases.sh >> ${WORKSPACE}/logs/backup.log 2>&1") | crontab -
```

Expected: Cron entry added. Verify with `crontab -l | grep backup-databases`.

If this fails: Check if `crontab` is available. On some systems, use `systemctl` timers or `launchd` plist instead (see Adaptation Points).

If already exists: Check if the existing schedule matches. If different, remove the old entry first then add the new one. If same, skip.

> **Idempotency:** Always check `crontab -l | grep backup-databases` before adding. Duplicate cron entries cause multiple concurrent backups that compete for Drive uploads.

#### 4.2 Create Log Directory `[AUTO]`

Ensure the log directory exists for cron output.

**Remote:**
```
mkdir -p ${WORKSPACE}/logs
```

Expected: Directory exists at `${WORKSPACE}/logs/`.

If already exists: `mkdir -p` is idempotent -- no action needed.

#### 4.3 Verify Failure Alerts `[GUIDED]`

Test that the backup script correctly sends failure alerts to Telegram. Simulate a failure by temporarily renaming the passphrase file.

**Remote:**
```
mv ${WORKSPACE}/.backup-passphrase ${WORKSPACE}/.backup-passphrase.test-bak
${WORKSPACE}/scripts/backup-databases.sh 2>&1 || true
mv ${WORKSPACE}/.backup-passphrase.test-bak ${WORKSPACE}/.backup-passphrase
```

Expected: The script fails at the encryption step and sends a "Backup FAILED" message to the Telegram cron-updates topic.

If this fails: The simplest end-to-end test is to run the backup script manually with the passphrase file removed (as above) and verify the Telegram alert arrives in the cron-updates topic. To test the channel plumbing in isolation, use the current `channels` API: `openclaw channels send --channel telegram --to "${TELEGRAM_GROUP}:${CRON_UPDATES_TOPIC_ID}" --message "test alert"`. If `channels` is not configured, the script falls back to raw Bot API — test that path with `curl -sS -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" --data-urlencode "chat_id=${TELEGRAM_GROUP}" --data-urlencode "message_thread_id=${CRON_UPDATES_TOPIC_ID}" --data-urlencode "text=test alert"`.

### Phase 5: Post-Install

#### 5.1 Update Client Profile `[AUTO]`

Update `clients/<name>.md` with:
- Database backup system deployed
- Backup schedule (hourly)
- Backup destination (Google Drive folder ID)
- Retention policy (last N backups)
- GPG passphrase location (`${WORKSPACE}/.backup-passphrase`)
- Restore command reference

#### 5.2 Note Follow-Ups `[AUTO]`

Tell the operator:
- **First hour:** Monitor the first automated backup by checking `${WORKSPACE}/logs/backup.log` after the top of the next hour
- **Restore documentation:** Remind the client that `${WORKSPACE}/scripts/restore-databases.sh --list` shows available backups and `--force` skips confirmation
- **Passphrase safety:** The GPG passphrase in `${WORKSPACE}/.backup-passphrase` should also be stored in a secure location (password manager, safe) in case the machine is lost

## Verification

Run these checks to confirm the backup system is fully operational:

**Verify backup script runs successfully:**

**Remote:**
```
${WORKSPACE}/scripts/backup-databases.sh
```

Expected: Completes with "Backup complete" message. No errors.

**Verify backup exists on Google Drive:**

**Remote:**
```
gog drive list --parent 'OpenClaw Backups' --limit 5
```

Expected: Encrypted archive and manifest files with recent timestamps.

**Verify restore can list backup contents:**

**Remote:**
```
${WORKSPACE}/scripts/restore-databases.sh --list
```

Expected: All databases listed with their original paths and sizes.

**Verify cron job is scheduled:**

**Remote:**
```
crontab -l | grep backup-databases
```

Expected: `0 * * * * ${WORKSPACE}/scripts/backup-databases.sh >> ${WORKSPACE}/logs/backup.log 2>&1` (with actual workspace path).

**Verify passphrase file is protected:**

**Remote:**
```
ls -la ${WORKSPACE}/.backup-passphrase
```

Expected: File permissions show `-rw-------` (mode 600, owner read/write only).

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Drive upload fails | `gog` auth expired or Drive scope missing | Re-authenticate: `gog auth login`. Verify Drive access: `gog drive list --limit 1`. |
| No databases found | Discovery paths don't match actual database locations | Check where databases actually live: `find ${WORKSPACE} ~/.openclaw -name '*.db' 2>/dev/null`. Update script paths. |
| GPG encryption errors | GPG not installed, passphrase file missing, or wrong permissions | Verify: `which gpg`, `test -f ${WORKSPACE}/.backup-passphrase`, `ls -la ${WORKSPACE}/.backup-passphrase`. |
| Restore fails | Manifest missing on Drive, or passphrase changed since backup was made | Download manifest manually with `gog drive list`. Old backups need the old passphrase. |
| Backup too large | Binary blobs, WAL files, or large JSONL logs included | Review discovered files. Exclude large non-essential files by adding exclusion patterns to the script. |
| Duplicate cron entries | Skill was run multiple times without checking for existing entries | `crontab -l` to see all entries. Remove duplicates: `crontab -l \| sort -u \| crontab -`. |
| Failure alert not sent | Telegram group/topic IDs wrong, or `openclaw channels` not configured | Test via the current API: `openclaw channels send --channel telegram --to "${TELEGRAM_GROUP}:${CRON_UPDATES_TOPIC_ID}" --message "test"`. If that fails, exercise the raw Bot API fallback: `curl -sS -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" --data-urlencode "chat_id=${TELEGRAM_GROUP}" --data-urlencode "message_thread_id=${CRON_UPDATES_TOPIC_ID}" --data-urlencode "text=test"`. |
| `wc -c` returns wrong value | macOS `wc -c` adds leading whitespace | The script uses `wc -c < file \| tr -d ' '` to handle this. If still wrong, use `stat -f%z file` (macOS). |
| Retention not cleaning up | `gog drive list` output format changed or folder has non-backup files | Check Drive folder contents manually. Clean up stale files. Review the retention logic in the script. |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Create Drive folder | Execute silently | Confirm before creating | Confirm |
| 1.2 Set encryption passphrase | Prompt for passphrase `[HUMAN_INPUT]` | Prompt for passphrase `[HUMAN_INPUT]` | Prompt for passphrase `[HUMAN_INPUT]` |
| 2.1 Write backup script | Execute silently | Show script, confirm before writing | Show script, confirm |
| 2.2 Write restore script | Execute silently | Show script, confirm before writing | Show script, confirm |
| 3.1 Manual backup test | Execute silently | Confirm before running | Confirm, review output |
| 3.2 Verify on Drive | Execute silently | Execute silently | Show output, confirm |
| 3.3 Test restore list | Execute silently | Execute silently | Show output, confirm |
| 4.1 Schedule cron | Execute silently | Confirm before adding | Confirm |
| 4.3 Test failure alert | Skip (optional) | Confirm before simulating failure | Confirm, review alert |
| 5.1 Update client profile | Execute silently | Execute silently | Show changes, confirm |

## Dependencies

- **Depends on:** `openclaw/install.md`, `deploy-google-workspace.md` (Drive access), `deploy-messaging-setup.md` (Telegram alerts)
- **Should deploy after:** Any database-creating system (CRM, knowledge base, social tracking, food journal, model tracking, etc.)
- **Required by:** None directly, but provides critical disaster recovery for all data systems
- **Related:** `deploy-git-autosync.md` (backs up code/config; this skill backs up binary database files which git cannot handle well)
