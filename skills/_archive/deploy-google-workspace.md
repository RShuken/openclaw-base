# Deploy Google Workspace Integration

## Purpose

Connect the agent to Google Workspace via the `gog` OAuth CLI. Provides Gmail, Calendar, Drive, and Docs/Sheets/Slides access needed by downstream systems: CRM contact discovery, urgent email detection, daily briefings, meeting pipelines, database backups, and on-demand document creation.

**When to use:** When setting up any system that needs email, calendar, file storage, or document creation access.

**When to skip:** If the client does not use Google Workspace at all. Downstream skills that depend on this (CRM email scanning, backups to Drive, calendar-aware briefings) have their own Adaptation Points tables with alternatives for non-Google providers.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "uname -m"
cmd --session <id> --command "which gog 2>/dev/null"
cmd --session <id> --command "gog auth status 2>/dev/null"
cmd --session <id> --command "ls ~/Library/Application\ Support/gogcli/ 2>/dev/null || ls ~/.config/gogcli/ 2>/dev/null"
```

Questions to answer before proceeding:
- What architecture is the machine? (`arm64` = Apple Silicon, `x86_64` = Intel Mac or Linux)
- Is `gog` already installed? If so, where? (`which gog`)
- Is there an existing OAuth session? (`gog auth status`)
- Does the client use Google Workspace, or different providers for email/calendar/storage?
- Which downstream systems will be deployed? (This determines which Google services are needed.)

## Adaptation Points

The recommended defaults below come from the reference implementation (Matt Berman's setup, battle-tested across 26 systems). Clients can use them as-is or customize any component.

| Component | Recommended Default | Customize When... |
|-----------|-------------------|-------------------|
| Email provider | Gmail via `gog` | Client uses Microsoft 365, IMAP, or another email provider |
| Calendar provider | Google Calendar via `gog` | Client uses Outlook Calendar, CalDAV, or another calendar |
| Cloud storage | Google Drive via `gog` | Client uses S3, Backblaze B2, Dropbox, or prefers local-only backups |
| Document creation | Google Docs/Sheets/Slides via `gog` | Client prefers Markdown, local files, or does not need cloud documents |
| OAuth CLI tool | `gog` installed via Homebrew | Linux (use `npm install`), or client already has a different Google OAuth tool |
| gog install path | Auto-detected by architecture (see table below) | Client has a custom install path or uses npm global |
| Backup folder name | "OpenClaw Backups" on Drive | Client wants a different name, or skip if not using Drive for backups |
| OAuth client | gog's built-in client ID | gog's client is blocked by Google ("not completed verification") — use Path B with user's own GCP project |
| OAuth scopes | All (Gmail + Calendar + Drive + Docs + Sheets + Slides) | Client only needs a subset of services |

**gog path by architecture:**

| Architecture | Expected gog Path | How to Detect |
|-------------|-------------------|---------------|
| Apple Silicon (arm64) | `/opt/homebrew/bin/gog` | `uname -m` returns `arm64` |
| Intel Mac (x86_64) | `/usr/local/bin/gog` | `uname -m` returns `x86_64` on macOS |
| Linux | Varies (npm global or Homebrew) | `which gog` after install |

## Prerequisites

- OpenClaw agent installed and running
- Homebrew available (macOS) or npm available (Linux) for installing `gog`
- Google account with the relevant services enabled (Gmail, Calendar, Drive as needed)
- Browser access on the client machine for the OAuth consent flow

## gog v0.9.0 CLI Reference

The actual gog v0.9.0 command syntax differs from what you might guess. Use these verified commands:

| Action | Correct Command | Wrong (will fail) |
|--------|----------------|-------------------|
| Search emails | `gog gmail search 'newer_than:1d' --plain` | ~~`gog gmail messages list --max-results 5`~~ |
| List calendar events | `gog calendar list --plain` | ~~`gog calendar events list --time-min ...`~~ |
| List Drive files | `gog drive ls --plain` | ~~`gog drive files list`~~ / ~~`gog drive list`~~ |
| Create Drive folder | `gog drive mkdir 'Folder Name' --plain` | ~~`gog drive folders create --name '...'`~~ |
| Check auth status | `gog auth status --plain` | (works as expected) |
| List authorized services | `gog auth services --plain` | (works as expected) |
| Import OAuth credentials | `gog auth credentials set <file.json> --client <name>` | (works as expected) |
| Authorize account | `gog auth add <email> --services=... --manual` | ~~`gog auth login`~~ |
| Get CLI help | `gog --help`, `gog <command> --help` | ~~`gog help <command>`~~ |

**Important:** When using a custom OAuth client, all gog commands require `--account <email> --client <name>` flags unless defaults are set via environment or config.

## What Gets Installed

| Component | Location / Path | Purpose |
|-----------|----------------|---------|
| `gog` CLI | Auto-detected (see architecture table above) | OAuth CLI for all Google services |
| OAuth tokens | `~/Library/Application Support/gogcli/` (macOS) or `~/.config/gogcli/` (Linux) | Persistent auth tokens (auto-refreshed) |
| Gmail access | via `gog gmail` | Email scanning for CRM contacts, urgent email detection, AI-assisted draft creation |
| Calendar access | via `gog calendar` | Meeting tracking, Fathom transcript trigger, daily briefing, double-booking detection |
| Drive access | via `gog drive` | Encrypted database backup destination, document storage for reports and exports |
| Docs/Sheets/Slides | via `gog docs` / `gog sheets` / `gog slides` | Create and share documents, spreadsheets, and presentations on demand |
| Backup folder | Google Drive (if Drive enabled) | Dedicated folder for automated hourly database backups |

## Steps

### 1. Check Existing Google Workspace Setup

Detect architecture and check for existing installation:

```
cmd --session <id> --command "uname -m"
cmd --session <id> --command "which gog 2>/dev/null"
cmd --session <id> --command "gog auth status 2>/dev/null"
```

If `gog` is already installed and authenticated:
- Confirm which scopes are granted
- Confirm the path matches the expected path for the architecture
- Skip to Step 4 (Verify Each Service)

If `gog` is installed but not authenticated:
- Skip to Step 4 (Run OAuth Login)

Check for an existing backup folder on Drive (if already authenticated):

```
cmd --session <id> --command "gog drive ls --plain 2>/dev/null | grep -i 'openclaw.*backup'"
```

### 2. Choose Which Google Services to Enable

Walk the client through which services they need. All four are recommended, but not all may be needed depending on which downstream systems will be deployed.

| Service | Recommended | What It Enables | Skip If... |
|---------|-------------|-----------------|------------|
| Gmail | Yes | CRM contact discovery, urgent email detection, AI-assisted draft creation, email context in briefings | Client uses non-Gmail email, or no email systems planned |
| Calendar | Yes | Meeting tracking, Fathom transcript trigger, attendee context in briefings, double-booking detection | Client uses non-Google calendar, or no meeting/briefing systems planned |
| Drive | Yes | Encrypted database backups, document storage for reports and exports | Client uses S3/Backblaze/local for backups and does not need cloud doc storage |
| Docs/Sheets/Slides | Yes | On-demand document, spreadsheet, and presentation creation and sharing | Client prefers Markdown or local-only documents |

Ask the client:
> "Google Workspace connects Gmail, Calendar, Drive, and Docs. The recommended setup enables all four -- Gmail feeds the CRM and email systems, Calendar feeds briefings and meeting pipelines, Drive stores encrypted backups, and Docs lets the agent create documents on demand. Want all four (recommended), or only specific services?"

Record the client's choices. The OAuth scopes in Step 4 and verification in Step 5 will adapt based on which services are enabled.

### 3. Install the gog CLI

If `gog` is already installed (detected in Step 1), skip this step.

**macOS (Homebrew -- recommended):**

```
cmd --session <id> --command "brew install drizzle-team/tap/gog"
```

Verify installation and detect the path:

```
cmd --session <id> --command "which gog && gog --version"
```

Expected paths:
- Apple Silicon (`arm64`): `/opt/homebrew/bin/gog`
- Intel Mac (`x86_64`): `/usr/local/bin/gog`

If the path does not match the expected path for the architecture, note the actual path for the config step.

**Linux (alternative):**

If Homebrew is not available, install via npm:

```
cmd --session <id> --command "npm install -g @drizzle-team/gog"
```

Verify:

```
cmd --session <id> --command "which gog && gog --version"
```

### 4. Run OAuth Login

There are two paths depending on whether the client can use gog's built-in OAuth client or needs their own.

#### Path A: gog's Default OAuth Client

gog ships with its own OAuth client ID. Try this first:

```
cmd --session <id> --command "gog auth add <email> --services=gmail,calendar,drive --manual --no-input 2>&1"
```

This prints an OAuth URL. Have the user open it in a browser. **Known issue:** gog's default OAuth client is in Google's "testing" mode. Only pre-approved test users can authorize. If the user sees **"Access blocked: gog CLI has not completed the Google verification process"**, they must use Path B instead.

#### Path B: Custom OAuth Client (Required for Most Users)

When gog's built-in client is blocked, the user creates their own OAuth credentials in Google Cloud Console:

**Step 1: User creates OAuth credentials (done by user in browser, ~5 minutes):**

1. Go to `console.cloud.google.com` → create or select a project
2. Enable APIs: Gmail API, Google Calendar API, Google Drive API
3. Go to **APIs & Services → OAuth consent screen** → set to **External**
4. **Add the user's email as a test user** (critical — without this, auth will fail with "access_denied")
5. Go to **APIs & Services → Credentials → Create Credentials → OAuth client ID**
6. Application type: **Desktop app**
7. Copy the **Client ID** and **Client Secret**

**Step 2: Import credentials into gog on the client machine:**

```
cmd --session <id> --command "cat > /tmp/gog-creds.json << 'EOF'
{\"installed\":{\"client_id\":\"<CLIENT_ID>\",\"client_secret\":\"<CLIENT_SECRET>\",\"auth_uri\":\"https://accounts.google.com/o/oauth2/auth\",\"token_uri\":\"https://oauth2.googleapis.com/token\",\"redirect_uris\":[\"http://localhost\"]}}
EOF
gog auth credentials set /tmp/gog-creds.json --client openclaw && rm /tmp/gog-creds.json"
```

**Step 3: Generate the OAuth URL:**

```
cmd --session <id> --command "gog auth add <email> --client openclaw --services=gmail,calendar,drive --force-consent --manual --no-input 2>&1"
```

This prints an OAuth URL. Send it to the user to open in their browser and authorize.

**Step 4: Complete the auth flow (remote operator console workaround):**

The operator console cannot handle interactive stdin, so gog's `--manual` mode requires a workaround. The user authorizes in their browser and gets redirected to a localhost URL that won't load. They copy the full redirect URL (contains `?state=...&code=...`).

**The problem:** gog generates a new CSRF state parameter each time it starts, so you cannot simply pipe the redirect URL from a previous invocation — the states won't match.

**The solution:** Use `expect` to start gog, capture its state, swap it into the redirect URL, and feed it back:

```
cmd --session <id> --command "cat > /tmp/gog-auth.expect << 'EXPECTEOF'
#!/usr/bin/expect -f
set timeout 30
set auth_code [lindex \$argv 0]

spawn gog auth add <email> --client openclaw --services=gmail,calendar,drive --force-consent --manual

expect {
    -re {state=([A-Za-z0-9_-]+)} {
        set gog_state \$expect_out(1,string)
    }
    timeout { puts \"ERROR: Could not capture state\"; exit 1 }
}

expect {
    \"Paste redirect URL\" {
        set redirect \"http://localhost:1/?state=\$gog_state&code=\$auth_code&scope=email%20https://www.googleapis.com/auth/drive%20https://www.googleapis.com/auth/calendar%20https://www.googleapis.com/auth/gmail.modify%20https://www.googleapis.com/auth/gmail.settings.basic%20https://www.googleapis.com/auth/gmail.settings.sharing%20https://www.googleapis.com/auth/userinfo.email%20openid&authuser=0&prompt=consent\"
        send \"\$redirect\r\"
    }
    timeout { puts \"ERROR: Timed out\"; exit 1 }
}

expect eof
set result [wait]
exit [lindex \$result 3]
EXPECTEOF
chmod +x /tmp/gog-auth.expect"
```

Then run it with the auth code extracted from the redirect URL:

```
cmd --session <id> --command "/usr/bin/expect /tmp/gog-auth.expect '<AUTH_CODE_FROM_REDIRECT_URL>' 2>&1"
```

Where `<AUTH_CODE_FROM_REDIRECT_URL>` is the `code=` parameter from the user's redirect URL.

**Verify auth succeeded:**

```
cmd --session <id> --command "gog --account <email> --client openclaw auth status --plain 2>&1"
```

Expected output should show `account: <email>`, `client: openclaw`, `credentials_exists: true`.

#### Setting gog Defaults

After auth, save defaults so downstream scripts can find them:

```
cmd --session <id> --command "
echo 'GOG_ACCOUNT=<email>' >> ${WORKSPACE}/.env
echo 'GOG_CLIENT=openclaw' >> ${WORKSPACE}/.env
echo 'GOG_CLI_PATH=$(which gog)' >> ${WORKSPACE}/.env
"
```

**Note:** gog's config.json does not reliably set default account/client. Always pass `--account` and `--client` flags explicitly in scripts and cron jobs, or source the workspace `.env` file.

### 5. Verify Each Enabled Service

Only verify the services the client chose in Step 2. Skip verification for services that were not enabled.

If using a custom OAuth client (Path B), add `--account <email> --client openclaw` to all gog commands.

**Gmail (if enabled):**

```
cmd --session <id> --command "gog gmail search 'newer_than:1d' --plain 2>&1 | head -5"
```

Expected: A TSV table with columns `ID, DATE, FROM, SUBJECT, LABELS, THREAD`. If this fails with a 403 permission error, the Gmail scope was not granted during OAuth -- re-run Step 4.

**Calendar (if enabled):**

```
cmd --session <id> --command "gog calendar list --plain 2>&1 | head -5"
```

Expected: A TSV table with columns `ID, START, END, SUMMARY`. An empty result is fine if no upcoming events. If it fails with `403 insufficientPermissions`, the Calendar scope was not granted.

**Drive (if enabled):**

```
cmd --session <id> --command "gog drive ls --plain 2>&1 | head -5"
```

Expected: A TSV table with columns `ID, NAME, TYPE, SIZE, MODIFIED`. If this fails with a 403 permission error, the Drive scope was not granted.

### 6. Create Backup Folder (If Drive Enabled)

Skip this step if Drive was not enabled in Step 2, or if the client uses a different backup destination.

Check if a backup folder already exists:

```
cmd --session <id> --command "gog drive ls --plain 2>&1 | grep -i 'openclaw.*backup'"
```

If it already exists, use the existing folder. Record its folder ID (first column) and skip to Step 7.

If it does not exist, create it:

```
cmd --session <id> --command "gog drive mkdir 'OpenClaw Backups' --plain 2>&1"
```

Expected output: a TSV line with `id`, `name`, and `link` fields. Record the folder ID.

Save the folder ID in the workspace environment:

```
cmd --session <id> --command "echo 'GOOGLE_DRIVE_BACKUP_FOLDER_ID=<folder-id>' >> ${WORKSPACE}/.env"
```

### 7. Record Config

Store the enabled services so other systems can find them. Most of this was already done in Step 4 (gog defaults) and Step 6 (backup folder ID). Verify the `.env` file has all required values:

```
cmd --session <id> --command "cat ${WORKSPACE}/.env"
```

Expected contents (values will vary per client):

```
GOG_ACCOUNT=<email>
GOG_CLIENT=<client-name>
GOG_CLI_PATH=<path-to-gog>
GOOGLE_DRIVE_BACKUP_FOLDER_ID=<folder-id>
GOOGLE_ENABLED_SERVICES=gmail,calendar,drive
```

Add any missing values:

```
cmd --session <id> --command "echo 'GOOGLE_ENABLED_SERVICES=gmail,calendar,drive' >> ${WORKSPACE}/.env"
```

Adjust the services list to match what was chosen in Step 2.

### 8. Update Client Profile

After deployment, update `clients/<name>.md` with:

```markdown
### Google Workspace
- **Status:** Deployed on [date]
- **Account:** [email used for Google auth]
- **gog path:** [actual path from `which gog`, e.g., /opt/homebrew/bin/gog]
- **gog client:** [client name, e.g., "openclaw" for custom, "default" for built-in]
- **Architecture:** [arm64 / x86_64]
- **Enabled services:** [Gmail, Calendar, Drive -- list only what was enabled]
- **Backup folder ID:** [folder ID, or "N/A -- using different backup destination"]
- **OAuth project:** [GCP project ID if custom client, or "gog default"]
- **OAuth scopes:** [list granted scopes]
- **Notes:** [Any issues encountered, custom folder names, or adaptations made]
```

## Verification

Run these commands to confirm the integration is working. Only check services that were enabled. If using a custom OAuth client, add `--account <email> --client <name>` to all gog commands.

**Auth status:**
```
cmd --session <id> --command "gog auth status --plain"
```
Expected: Shows `account`, `client`, `credentials_exists: true`.

**Authorized services:**
```
cmd --session <id> --command "gog auth services --plain"
```
Expected: TSV table showing `gmail`, `calendar`, `drive` with `USER: true`.

**Gmail (if enabled):**
```
cmd --session <id> --command "gog gmail search 'newer_than:1d' --plain 2>&1 | head -3"
```
Expected: At least one email row with columns `ID, DATE, FROM, SUBJECT, LABELS, THREAD`.

**Calendar (if enabled):**
```
cmd --session <id> --command "gog calendar list --plain 2>&1 | head -3"
```
Expected: Upcoming events (or empty result if none scheduled — not an error).

**Drive (if enabled):**
```
cmd --session <id> --command "gog drive ls --plain 2>&1 | head -3"
```
Expected: At least one file row with columns `ID, NAME, TYPE, SIZE, MODIFIED`.

**Config:**
```
cmd --session <id> --command "cat ${WORKSPACE}/.env"
```
Expected: `GOG_CLI_PATH`, `GOG_ACCOUNT`, `GOG_CLIENT`, `GOOGLE_ENABLED_SERVICES`, and `GOOGLE_DRIVE_BACKUP_FOLDER_ID` (if Drive enabled) are set.

**gog path sanity check:**
```
cmd --session <id> --command "test -x $(which gog) && echo 'gog is executable' || echo 'gog NOT FOUND or not executable'"
```

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| **"Access blocked: gog CLI has not completed the Google verification process"** | gog's built-in OAuth client is in Google's "testing" mode and the user's email is not an approved test user | Use Path B: create a custom OAuth client in the user's own GCP project. The user must also add their email as a **test user** in the OAuth consent screen. |
| **"Access blocked: ... can only be accessed by developer-approved testers"** (no "Advanced" bypass link) | The OAuth app is in "testing" mode with no bypass available (Google removed the "Advanced → Go to unsafe" option for testing-mode apps) | The user MUST be added as a test user in the GCP project's OAuth consent screen → Audience → Test users. There is no way around this. |
| **403 insufficientPermissions on Calendar or Drive** | The OAuth token was authorized with only Gmail scopes (partial auth) | Re-authorize with broader scopes: `gog auth add <email> --services=gmail,calendar,drive --force-consent --manual`. The `--force-consent` flag is required to re-prompt for all scopes. |
| **"state mismatch" when piping redirect URL to gog** | gog generates a new CSRF state parameter each invocation, so a redirect URL from a different session will fail validation | Use the `expect` workaround in Step 4 Path B: start gog, capture its state, swap it into the redirect URL, feed it back to the same process. |
| **gog command syntax errors** (e.g., "unexpected argument list, did you mean 'ls'?") | gog v0.9.0 command syntax differs from what you might guess. `drive list` → `drive ls`, `drive folders create` → `drive mkdir`, `gmail messages list` → `gmail search`, etc. | Refer to the **gog v0.9.0 CLI Reference** table above for correct syntax. Use `gog <command> --help` to discover subcommands. |
| **gog config.json doesn't set default account/client** | gog's config file format does not support `default_account` or `default_client` fields (as of v0.9.0) | Always pass `--account <email> --client <name>` flags explicitly. Store these in `${WORKSPACE}/.env` and source it from scripts/cron jobs. |
| `gog` command not found | Homebrew not in PATH or installation failed | Check `brew --version`. On Apple Silicon, verify `/opt/homebrew/bin/` is in PATH. On Intel Mac, verify `/usr/local/bin/` is in PATH. Reinstall with `brew install drizzle-team/tap/gog`. |
| gog installed at wrong path | Architecture mismatch or mixed Homebrew install | Run `uname -m` to confirm architecture. Apple Silicon should use `/opt/homebrew/bin/gog`, Intel Mac should use `/usr/local/bin/gog`. Reinstall if path is unexpected. |
| OAuth token expired | Tokens expire after a period of inactivity | Re-authorize: `gog auth add <email> --client <name> --services=gmail,calendar,drive --manual`. |
| Browser does not open for OAuth | Headless environment, SSH session, or remote operator console | Use `--manual` flag. gog prints a URL to visit in any browser. After authorizing, copy the redirect URL from the browser's address bar. |
| Linux: Homebrew not available | Different package manager | Install via npm: `npm install -g @drizzle-team/gog`. |
| Token refresh fails silently | Corrupted token store | Delete `~/Library/Application Support/gogcli/keyring/` (macOS) or `~/.config/gogcli/keyring/` (Linux) and re-authenticate. |
| Backup folder already exists | Previous deployment or manual creation | Use the existing folder. Run `gog drive ls --plain \| grep -i backup` to find the folder ID. |
| Intel Mac: Homebrew path issues | Homebrew installs to `/usr/local/` on Intel, not `/opt/homebrew/` | Confirm with `uname -m`. If `x86_64`, gog should be at `/usr/local/bin/gog`. |
| Cron jobs cannot find gog | PATH not set in cron environment | Source `${WORKSPACE}/.env` at the top of cron scripts to get `GOG_CLI_PATH`, `GOG_ACCOUNT`, and `GOG_CLIENT`. |
| Python SSL cert error on macOS | System Python's SSL certs are outdated or missing | Use `curl` instead of Python's `urllib` for API calls. Or install `certifi`: `pip3 install certifi`. |

## Dependencies

- **Depends on:** Nothing directly, but requires a Google account with the relevant services enabled. Agent must be installed and running.
- **Required by:**
  - `deploy-personal-crm.md` (Gmail scanning for contacts, Calendar for meeting tracking)
  - `deploy-urgent-email.md` (Gmail monitoring for urgent email classification)
  - `deploy-daily-briefing.md` (Calendar events for morning briefing, email context)
  - `deploy-db-backups.md` (Drive as encrypted backup destination)
  - `deploy-fathom-pipeline.md` (Calendar events trigger transcript processing)
- **Optional for:**
  - `deploy-advisory-council.md` (email as a data source for analysis)
  - `deploy-earnings.md` (Gmail receipts and invoices)
  - `deploy-video-pipeline.md` (Drive for document storage)
