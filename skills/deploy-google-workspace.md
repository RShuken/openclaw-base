---
name: "Deploy Google Workspace Integration"
category: "integration"
subcategory: "productivity"
third_party_service: "Google Workspace (Gmail, Calendar, Drive)"
auth_type: "oauth"
openclaw_version_required: "2026.4.15+"
version_last_verified: "2026-04-20"
last_checked: "2026-04-20"
maintenance_status: "active"
model_env_var: "n/a"
clawhub_alternative: "@steipete/gog"
cost_tier: "free"
privacy_tier: "cloud"
requires:
  - "Google account"
  - "gog CLI (Homebrew on macOS, npm on Linux)"
  - "Google OAuth consent"
docs_urls:
  - "https://github.com/steipete/gog"
  - "https://developers.google.com/workspace"
---

# Deploy Google Workspace Integration

## Compatibility

- **OpenClaw Version**: 2026.4.15+ (compat header rewritten 2026-04-20 — live-verified on 4C)
- **Status**: WORKING — no `openclaw` CLI dependencies (uses `gog` CLI by @steipete throughout)
- **Known issue**: remote OAuth flow has a state-mismatch bug — use the `expect` workaround documented in the skill body. `gog auth keyring file` should be called first to avoid macOS Keychain prompts.
- **ClawHub alternative**: `@steipete/gog` (158k downloads per `audit/_baseline.md` §B) — this skill essentially wraps that CLI's install + OAuth flow. Consider installing the ClawHub skill directly as a simpler path.
- **Model**: agent default (Codex-first); do not hardcode

## Purpose

Connect the agent to Google Workspace via the `gog` OAuth CLI. Provides Gmail, Calendar, Drive, and Docs/Sheets/Slides access needed by downstream systems: CRM contact discovery, urgent email detection, daily briefings, meeting pipelines, database backups, and on-demand document creation.

**When to use:** When setting up any system that needs email, calendar, file storage, or document creation access.

**When to skip:** If the client does not use Google Workspace at all. Downstream skills that depend on this (CRM email scanning, backups to Drive, calendar-aware briefings) have their own Adaptation Points tables with alternatives for non-Google providers.

**What this skill does:**
1. Checks for existing gog installation and OAuth session
2. Installs the `gog` CLI (Homebrew on macOS, npm on Linux)
3. Runs the OAuth flow to authorize Google services (Gmail, Calendar, Drive)
4. Verifies each enabled service works
5. Creates a backup folder on Drive (if Drive is enabled)
6. Records config in the workspace `.env` file

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

Resolve these before executing. Sources are listed for each.

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${CLIENT_EMAIL}` | Client profile or ask client: Google account email | `user@gmail.com` |
| `${GOG_CLIENT_NAME}` | `"default"` for built-in client, or a name like `"openclaw"` for custom | `openclaw` |
| `${GCP_CLIENT_ID}` | User's GCP OAuth credentials (Path B only) | `123456-abc.apps.googleusercontent.com` |
| `${GCP_CLIENT_SECRET}` | User's GCP OAuth credentials (Path B only) | `GOCSPX-abc123...` |
| `${GOG_PATH}` | Detected from `which gog` on client machine | `/opt/homebrew/bin/gog` |
| `${DRIVE_BACKUP_FOLDER_ID}` | Response from Drive mkdir command (Phase 4) | `1abc2def3ghi...` |
| `${ENABLED_SERVICES}` | Client choice: which Google services to authorize | `gmail,calendar,drive` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**

Detect architecture:

**Remote:**
```
uname -m
```

Check if gog is already installed:

**Remote:**
```
which gog 2>/dev/null || echo 'NOT_INSTALLED'
```

Check for existing OAuth session:

**Remote:**
```
gog auth status 2>/dev/null || echo 'NOT_AUTHENTICATED'
```

Check for existing gog config directory:

**Remote (macOS):**
```
ls ~/Library/Application\ Support/gogcli/ 2>/dev/null || echo 'NO_CONFIG_DIR'
```

**Remote (Linux):**
```
ls ~/.config/gogcli/ 2>/dev/null || echo 'NO_CONFIG_DIR'
```

Check if `expect` is available (needed for OAuth flow workaround):

**Remote:**
```
which expect 2>/dev/null || echo 'EXPECT_NOT_INSTALLED'
```

**Decision points from pre-flight:**
- What architecture is the machine? (`arm64` = Apple Silicon, `x86_64` = Intel Mac or Linux)
- Is `gog` already installed? If so, where? (sets `${GOG_PATH}`)
- Is there an existing OAuth session? (`gog auth status`)
- Is `expect` installed? (Required for Path B OAuth flow. On macOS it is pre-installed. On Linux, install via `apt install expect` or `yum install expect`.)
- Does the client use Google Workspace, or different providers for email/calendar/storage?
- Which downstream systems will be deployed? (This determines which Google services are needed.)

## Adaptation Points

The recommended defaults below come from the reference implementation. Clients can use them as-is or customize any component.

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Email provider | Gmail via `gog` | Client uses Microsoft 365, IMAP, or another email provider |
| Calendar provider | Google Calendar via `gog` | Client uses Outlook Calendar, CalDAV, or another calendar |
| Cloud storage | Google Drive via `gog` | Client uses S3, Backblaze B2, Dropbox, or prefers local-only backups |
| Document creation | Google Docs/Sheets/Slides via `gog` | Client prefers Markdown, local files, or does not need cloud documents |
| OAuth CLI tool | `gog` installed via Homebrew | Linux (use `npm install`), or client already has a different Google OAuth tool |
| gog install path | Auto-detected by architecture (see table below) | Client has a custom install path or uses npm global |
| Backup folder name | "OpenClaw Backups" on Drive | Client wants a different name, or skip if not using Drive for backups |
| OAuth client | gog's built-in client ID | gog's client is blocked by Google ("not completed verification") -- use Path B with user's own GCP project |
| OAuth scopes | All (Gmail + Calendar + Drive + Docs + Sheets + Slides) | Client only needs a subset of services |

**gog path by architecture:**

| Architecture | Expected gog Path | How to Detect |
|-------------|-------------------|---------------|
| Apple Silicon (arm64) | `/opt/homebrew/bin/gog` | `uname -m` returns `arm64` |
| Intel Mac (x86_64) | `/usr/local/bin/gog` | `uname -m` returns `x86_64` on macOS |
| Linux | Varies (npm global or Homebrew) | `which gog` after install |

## Prerequisites

- OpenClaw agent installed and running (`openclaw/install.md`)
- Homebrew available (macOS) or npm available (Linux) for installing `gog`
- `expect` installed on the client machine (pre-installed on macOS; on Linux: `apt install expect` or `yum install expect`)
- Google account with the relevant services enabled (Gmail, Calendar, Drive as needed)
- Browser access on the client machine (or any device) for the OAuth consent flow

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

### Phase 1: Check Existing Setup `[AUTO]`

#### 1.1 Detect Architecture and Existing Installation

Check the client's architecture, whether gog is already installed, and whether an OAuth session exists.

**Remote:**
```
uname -m
```

**Remote:**
```
which gog 2>/dev/null || echo 'NOT_INSTALLED'
```

**Remote:**
```
gog auth status 2>/dev/null || echo 'NOT_AUTHENTICATED'
```

Expected: Architecture string (e.g., `arm64` or `x86_64`), gog path or `NOT_INSTALLED`, and auth status or `NOT_AUTHENTICATED`.

If gog is already installed and authenticated:
- Confirm which scopes are granted
- Confirm the path matches the expected path for the architecture
- Set `${GOG_PATH}` from the `which gog` output
- Skip to Phase 3 (Verify Each Service)

If gog is installed but not authenticated:
- Set `${GOG_PATH}` and skip to Phase 2 Step 4 (Run OAuth Login)

#### 1.2 Check for Existing Backup Folder on Drive `[AUTO]`

If gog is already authenticated, check whether a backup folder already exists.

**Remote:**
```
gog drive ls --plain 2>/dev/null | grep -i 'openclaw.*backup'
```

Expected: Either a TSV row with the existing folder (record its ID), or no output (folder does not exist yet).

If already exists: Record the folder ID (first column) and skip folder creation in Phase 4.

#### 1.3 Check for expect `[AUTO]`

Verify that `expect` is installed (needed for the OAuth workaround in Phase 2).

**Remote:**
```
which expect 2>/dev/null || echo 'EXPECT_NOT_INSTALLED'
```

Expected: Path to expect (e.g., `/usr/bin/expect`).

If this fails: Install expect before proceeding to Phase 2.
- macOS: Pre-installed. If missing, `brew install expect`.
- Debian/Ubuntu: `sudo apt install -y expect`
- RHEL/CentOS: `sudo yum install -y expect`

### Phase 2: Install and Authenticate `[GUIDED]`

#### 2.1 Choose Which Google Services to Enable `[HUMAN_INPUT]`

Walk the client through which services they need. All four are recommended, but not all may be needed depending on which downstream systems will be deployed.

| Service | Recommended | What It Enables | Skip If... |
|---------|-------------|-----------------|------------|
| Gmail | Yes | CRM contact discovery, urgent email detection, AI-assisted draft creation, email context in briefings | Client uses non-Gmail email, or no email systems planned |
| Calendar | Yes | Meeting tracking, Fathom transcript trigger, attendee context in briefings, double-booking detection | Client uses non-Google calendar, or no meeting/briefing systems planned |
| Drive | Yes | Encrypted database backups, document storage for reports and exports | Client uses S3/Backblaze/local for backups and does not need cloud doc storage |
| Docs/Sheets/Slides | Yes | On-demand document, spreadsheet, and presentation creation and sharing | Client prefers Markdown or local-only documents |

Ask the client:
> "Google Workspace connects Gmail, Calendar, Drive, and Docs. The recommended setup enables all four -- Gmail feeds the CRM and email systems, Calendar feeds briefings and meeting pipelines, Drive stores encrypted backups, and Docs lets the agent create documents on demand. Want all four (recommended), or only specific services?"

Record the client's choices as `${ENABLED_SERVICES}` (e.g., `gmail,calendar,drive`). The OAuth scopes and verification steps adapt based on this.

#### 2.2 Install the gog CLI `[GUIDED]`

If gog was already detected in Phase 1, skip this step.

Install gog via Homebrew (macOS) or npm (Linux).

**Remote (macOS -- Homebrew):**
```
brew install drizzle-team/tap/gog
```

**Remote (Linux -- npm alternative):**
```
npm install -g @drizzle-team/gog
```

Verify installation and detect the path:

**Remote:**
```
which gog && gog --version
```

Expected: Path to gog binary and version number. Expected paths:
- Apple Silicon (`arm64`): `/opt/homebrew/bin/gog`
- Intel Mac (`x86_64`): `/usr/local/bin/gog`
- Linux: varies (typically in npm global bin)

Set `${GOG_PATH}` from the `which gog` output. If the path does not match the expected path for the architecture, note the actual path.

If this fails: On macOS, check `brew --version` to confirm Homebrew is installed. On Apple Silicon, verify `/opt/homebrew/bin/` is in PATH. On Intel Mac, verify `/usr/local/bin/` is in PATH.

If already installed: Skip. Note the existing path.

#### 2.3 Attempt Default OAuth Client (Path A) `[GUIDED]`

gog ships with its own OAuth client ID. Try this first.

Start the OAuth flow with the built-in client. The `--manual` and `--no-input` flags prevent gog from trying to open a browser or read stdin.

**Remote:**
```
gog auth add ${CLIENT_EMAIL} --services=${ENABLED_SERVICES} --manual --no-input 2>&1
```

Expected: An OAuth URL is printed. Have the user open it in a browser.

**Known issue:** gog's default OAuth client is in Google's "testing" mode. Only pre-approved test users can authorize. If the user sees **"Access blocked: gog CLI has not completed the Google verification process"**, they must use Path B (Step 2.4) instead.

If Path A succeeds (user authorizes and gog confirms), skip to Step 2.6.

#### 2.4 Custom OAuth Client Setup (Path B) `[HUMAN_INPUT]`

When gog's built-in client is blocked, the user creates their own OAuth credentials in Google Cloud Console.

**Step 1: User creates OAuth credentials (done by user in browser, ~5 minutes):**

1. Go to `console.cloud.google.com` -> create or select a project
2. Enable APIs: Gmail API, Google Calendar API, Google Drive API
3. Go to **APIs & Services -> OAuth consent screen** -> set to **External**
4. **Add the user's email as a test user** (critical -- without this, auth will fail with "access_denied")
5. Go to **APIs & Services -> Credentials -> Create Credentials -> OAuth client ID**
6. Application type: **Desktop app**
7. Copy the **Client ID** and **Client Secret**

Record these as `${GCP_CLIENT_ID}` and `${GCP_CLIENT_SECRET}`.

**Step 2: Write credentials file and import into gog.**

Write the OAuth credentials JSON file to the client machine. The operator must substitute `${GCP_CLIENT_ID}` and `${GCP_CLIENT_SECRET}` with the actual values before sending the command.

**Remote:**
```
cat > /tmp/gog-creds.json << EOF
{"installed":{"client_id":"${GCP_CLIENT_ID}","client_secret":"${GCP_CLIENT_SECRET}","auth_uri":"https://accounts.google.com/o/oauth2/auth","token_uri":"https://oauth2.googleapis.com/token","redirect_uris":["http://localhost"]}}
EOF
```

> **Important:** This uses an unquoted heredoc (`EOF`, not `'EOF'`) so that the operator's variable substitution produces valid JSON. The variables must be resolved to literal values before sending through the session API. The resulting file must contain actual client_id and client_secret strings, not placeholder text.

Then import the credentials:

**Remote:**
```
gog auth credentials set /tmp/gog-creds.json --client ${GOG_CLIENT_NAME} && rm /tmp/gog-creds.json
```

Expected: No error output. The credentials are imported into gog's config.

If this fails: Verify the JSON file was written correctly: `cat /tmp/gog-creds.json | python3 -m json.tool`. Common cause: malformed JSON from incorrect variable substitution.

#### 2.5 Complete OAuth Flow (Path B) `[HUMAN_INPUT]`

**Step 1: Generate the OAuth URL.**

**Remote:**
```
gog auth add ${CLIENT_EMAIL} --client ${GOG_CLIENT_NAME} --services=${ENABLED_SERVICES} --force-consent --manual --no-input 2>&1
```

Expected: An OAuth URL is printed. Send it to the user to open in their browser and authorize.

**Step 2: User authorizes and captures the redirect URL.**

The user authorizes in their browser and gets redirected to a localhost URL that will not load. They copy the full redirect URL from their browser's address bar (contains `?state=...&code=...`).

Extract the `code=` parameter value from the redirect URL. This is `${AUTH_CODE}`.

**Step 3: Complete the auth flow using expect.**

The operator console cannot handle interactive stdin, so gog's `--manual` mode requires a workaround. The problem: gog generates a new CSRF state parameter each time it starts, so you cannot pipe a redirect URL from a different invocation -- the states will not match.

The solution: Use `expect` to start gog, capture its state parameter, construct a redirect URL with the matching state, and feed it back to the same gog process.

Write the expect script. Note: this uses an **unquoted heredoc** so that `${CLIENT_EMAIL}`, `${GOG_CLIENT_NAME}`, and `${ENABLED_SERVICES}` are substituted with their literal values before sending. The Tcl variables (`$argv`, `$expect_out`, `$gog_state`, `$auth_code`, `$redirect`) must remain as-is for expect/Tcl to interpret them at runtime.

**Remote:**
```
cat > /tmp/gog-auth.expect << EOF
#!/usr/bin/expect -f
set timeout 30
set auth_code [lindex \$argv 0]

spawn gog auth add ${CLIENT_EMAIL} --client ${GOG_CLIENT_NAME} --services=${ENABLED_SERVICES} --force-consent --manual

expect {
    -re {state=([A-Za-z0-9_-]+)} {
        set gog_state \$expect_out(1,string)
    }
    timeout { puts "ERROR: Could not capture state"; exit 1 }
}

expect {
    "Paste redirect URL" {
        set redirect "http://localhost:1/?state=\$gog_state&code=\$auth_code&scope=email%20https://www.googleapis.com/auth/drive%20https://www.googleapis.com/auth/calendar%20https://www.googleapis.com/auth/gmail.modify%20https://www.googleapis.com/auth/gmail.settings.basic%20https://www.googleapis.com/auth/gmail.settings.sharing%20https://www.googleapis.com/auth/userinfo.email%20openid&authuser=0&prompt=consent"
        send "\$redirect\r"
    }
    timeout { puts "ERROR: Timed out"; exit 1 }
}

expect eof
set result [wait]
exit [lindex \$result 3]
EOF
chmod +x /tmp/gog-auth.expect
```

> **Critical syntax note:** This heredoc uses unquoted `EOF` (not `'EOF'`). The `${CLIENT_EMAIL}`, `${GOG_CLIENT_NAME}`, and `${ENABLED_SERVICES}` placeholders get substituted by the operator before sending (they become literal values in the script). The Tcl variable references (`$argv`, `$expect_out`, `$gog_state`, `$auth_code`, `$redirect`) are escaped with `\$` so they survive the shell's expansion and remain as `$variable` for Tcl/expect to interpret at runtime.

Then run the expect script with the auth code:

**Remote:**
```
/usr/bin/expect /tmp/gog-auth.expect '${AUTH_CODE}' 2>&1
```

Where `${AUTH_CODE}` is the `code=` parameter value extracted from the user's redirect URL. Substitute the actual auth code before sending.

Expected: gog completes the OAuth token exchange and confirms authorization.

If this fails:
- "state mismatch": The expect script did not capture the state correctly. Check the output for the OAuth URL and verify the regex matched.
- "ERROR: Could not capture state": gog may have changed its output format. Run `gog auth add ... --manual --no-input` manually and inspect the output to find the state parameter.
- "ERROR: Timed out": gog may be waiting for a different prompt. Increase the timeout or check `gog --help` for changes to the manual auth flow.
- Auth code expired: OAuth codes expire quickly (~5 minutes). Have the user re-authorize and get a fresh code.

#### 2.6 Verify Auth Succeeded `[AUTO]`

Confirm the OAuth session is active.

**Remote:**
```
gog auth status --plain 2>&1
```

If using a custom OAuth client (Path B):

**Remote:**
```
gog --account ${CLIENT_EMAIL} --client ${GOG_CLIENT_NAME} auth status --plain 2>&1
```

Expected: Output shows `account: ${CLIENT_EMAIL}`, `client: ${GOG_CLIENT_NAME}` (or `default`), `credentials_exists: true`.

If this fails: The OAuth flow did not complete. Return to Step 2.3 (Path A) or Step 2.5 (Path B) and retry.

#### 2.7 Save gog Defaults to Workspace .env `[AUTO]`

Store gog configuration so downstream scripts can find them. The operator must construct the command with literal values substituted.

Write the gog account name:

**Remote:**
```
echo "GOG_ACCOUNT=${CLIENT_EMAIL}" >> ${WORKSPACE}/.env
```

Write the gog client name:

**Remote:**
```
echo "GOG_CLIENT=${GOG_CLIENT_NAME}" >> ${WORKSPACE}/.env
```

Write the gog CLI path. This needs to resolve `which gog` on the client machine so the actual path is stored (not a literal command substitution string).

**Remote:**
```
echo "GOG_CLI_PATH=$(which gog)" >> ${WORKSPACE}/.env
```

> **Important:** This command uses double quotes so that the shell on the client machine executes `$(which gog)` and writes the resolved path (e.g., `GOG_CLI_PATH=/opt/homebrew/bin/gog`). If single quotes were used, the literal string `$(which gog)` would be written to the file, which is incorrect.

Expected: Three lines appended to `${WORKSPACE}/.env`.

If this fails: Check that `${WORKSPACE}/.env` exists and is writable. Create it first if needed: `touch ${WORKSPACE}/.env`.

If already exists: Check if the values are already present (e.g., `grep GOG_ACCOUNT ${WORKSPACE}/.env`). If they exist with correct values, skip. If they exist with wrong values, remove the old lines first: `sed -i '' '/^GOG_ACCOUNT=/d; /^GOG_CLIENT=/d; /^GOG_CLI_PATH=/d' ${WORKSPACE}/.env` (macOS) then re-append.

### Phase 3: Verify Each Enabled Service `[AUTO]`

Only verify the services the client chose in Step 2.1. Skip verification for services that were not enabled.

If using a custom OAuth client (Path B), add `--account ${CLIENT_EMAIL} --client ${GOG_CLIENT_NAME}` to all gog commands.

#### 3.1 Verify Gmail `[AUTO]`

Skip if Gmail was not enabled.

**Remote:**
```
gog gmail search 'newer_than:1d' --plain 2>&1 | head -5
```

Expected: A TSV table with columns `ID, DATE, FROM, SUBJECT, LABELS, THREAD`.

If this fails: A 403 permission error means the Gmail scope was not granted during OAuth. Re-run the OAuth flow (Phase 2) with `--force-consent` to re-prompt for all scopes.

#### 3.2 Verify Calendar `[AUTO]`

Skip if Calendar was not enabled.

**Remote:**
```
gog calendar list --plain 2>&1 | head -5
```

Expected: A TSV table with columns `ID, START, END, SUMMARY`. An empty result is fine if no upcoming events -- that is not an error.

If this fails: `403 insufficientPermissions` means the Calendar scope was not granted. Re-authorize with broader scopes: `gog auth add ${CLIENT_EMAIL} --services=${ENABLED_SERVICES} --force-consent --manual`.

#### 3.3 Verify Drive `[AUTO]`

Skip if Drive was not enabled.

**Remote:**
```
gog drive ls --plain 2>&1 | head -5
```

Expected: A TSV table with columns `ID, NAME, TYPE, SIZE, MODIFIED`.

If this fails: A 403 permission error means the Drive scope was not granted. Re-authorize with `--force-consent`.

### Phase 4: Create Backup Folder `[GUIDED]`

Skip this entire phase if Drive was not enabled, or if the client uses a different backup destination.

#### 4.1 Check for Existing Backup Folder `[AUTO]`

**Remote:**
```
gog drive ls --plain 2>&1 | grep -i 'openclaw.*backup'
```

Expected: Either a TSV row with the existing folder, or no output.

If already exists: Record the folder ID (first column of the TSV row) as `${DRIVE_BACKUP_FOLDER_ID}`. Skip to Step 4.3.

#### 4.2 Create Backup Folder `[GUIDED]`

**Remote:**
```
gog drive mkdir 'OpenClaw Backups' --plain 2>&1
```

Expected: A TSV line with `id`, `name`, and `link` fields. Record the folder ID as `${DRIVE_BACKUP_FOLDER_ID}`.

If this fails: Check Drive permissions (Phase 3 verification should have caught this). Also check for network issues.

#### 4.3 Save Backup Folder ID `[AUTO]`

**Remote:**
```
echo "GOOGLE_DRIVE_BACKUP_FOLDER_ID=${DRIVE_BACKUP_FOLDER_ID}" >> ${WORKSPACE}/.env
```

Expected: Line appended to `${WORKSPACE}/.env`.

If already exists: Check if the value is already present and correct. Skip if unchanged.

### Phase 5: Finalize Configuration `[AUTO]`

#### 5.1 Record Enabled Services `[AUTO]`

**Remote:**
```
echo "GOOGLE_ENABLED_SERVICES=${ENABLED_SERVICES}" >> ${WORKSPACE}/.env
```

Expected: Line appended to `${WORKSPACE}/.env`.

If already exists: Check the existing value. Update if the set of enabled services has changed.

#### 5.2 Verify Complete .env `[AUTO]`

Read the full `.env` file to confirm all expected values are present.

**Remote:**
```
cat ${WORKSPACE}/.env
```

Expected contents (values will vary per client):

```
GOG_ACCOUNT=user@gmail.com
GOG_CLIENT=openclaw
GOG_CLI_PATH=/opt/homebrew/bin/gog
GOOGLE_DRIVE_BACKUP_FOLDER_ID=1abc2def3ghi...
GOOGLE_ENABLED_SERVICES=gmail,calendar,drive
```

If values are missing, add them with the appropriate `echo "KEY=VALUE" >> ${WORKSPACE}/.env` command.

#### 5.3 Update Client Profile `[AUTO]`

Update `clients/<name>.md` with:

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

Run these commands to confirm the integration is working. Only check services that were enabled. If using a custom OAuth client, add `--account ${CLIENT_EMAIL} --client ${GOG_CLIENT_NAME}` to all gog commands.

**Auth status:**

**Remote:**
```
gog auth status --plain
```
Expected: Shows `account`, `client`, `credentials_exists: true`.

**Authorized services:**

**Remote:**
```
gog auth services --plain
```
Expected: TSV table showing `gmail`, `calendar`, `drive` with `USER: true`.

**Gmail (if enabled):**

**Remote:**
```
gog gmail search 'newer_than:1d' --plain 2>&1 | head -3
```
Expected: At least one email row with columns `ID, DATE, FROM, SUBJECT, LABELS, THREAD`.

**Calendar (if enabled):**

**Remote:**
```
gog calendar list --plain 2>&1 | head -3
```
Expected: Upcoming events (or empty result if none scheduled -- not an error).

**Drive (if enabled):**

**Remote:**
```
gog drive ls --plain 2>&1 | head -3
```
Expected: At least one file row with columns `ID, NAME, TYPE, SIZE, MODIFIED`.

**Config completeness:**

**Remote:**
```
cat ${WORKSPACE}/.env
```
Expected: `GOG_CLI_PATH`, `GOG_ACCOUNT`, `GOG_CLIENT`, `GOOGLE_ENABLED_SERVICES`, and `GOOGLE_DRIVE_BACKUP_FOLDER_ID` (if Drive enabled) are set.

**gog path sanity check:**

**Remote:**
```
test -x "$(which gog)" && echo 'gog is executable' || echo 'gog NOT FOUND or not executable'
```

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| **"Access blocked: gog CLI has not completed the Google verification process"** | gog's built-in OAuth client is in Google's "testing" mode and the user's email is not an approved test user | Use Path B: create a custom OAuth client in the user's own GCP project. The user must also add their email as a **test user** in the OAuth consent screen. |
| **"Access blocked: ... can only be accessed by developer-approved testers"** (no "Advanced" bypass link) | The OAuth app is in "testing" mode with no bypass available | The user MUST be added as a test user in the GCP project's OAuth consent screen -> Audience -> Test users. There is no way around this. |
| **403 insufficientPermissions on Calendar or Drive** | The OAuth token was authorized with only Gmail scopes (partial auth) | Re-authorize with broader scopes: `gog auth add <email> --services=gmail,calendar,drive --force-consent --manual`. The `--force-consent` flag is required to re-prompt for all scopes. |
| **"state mismatch" when piping redirect URL to gog** | gog generates a new CSRF state parameter each invocation, so a redirect URL from a different session will fail validation | Use the `expect` workaround in Step 2.5: start gog, capture its state, swap it into the redirect URL, feed it back to the same process. |
| **gog command syntax errors** (e.g., "unexpected argument list, did you mean 'ls'?") | gog v0.9.0 command syntax differs from what you might guess | Refer to the **gog v0.9.0 CLI Reference** table above for correct syntax. Use `gog <command> --help` to discover subcommands. |
| **gog config.json doesn't set default account/client** | gog's config file format does not support `default_account` or `default_client` fields (as of v0.9.0) | Always pass `--account <email> --client <name>` flags explicitly. Store these in `${WORKSPACE}/.env` and source it from scripts/cron jobs. |
| `gog` command not found | Homebrew not in PATH or installation failed | Check `brew --version`. On Apple Silicon, verify `/opt/homebrew/bin/` is in PATH. On Intel Mac, verify `/usr/local/bin/` is in PATH. Reinstall with `brew install drizzle-team/tap/gog`. |
| `expect` command not found | expect not installed (Linux) | Install: `apt install expect` (Debian/Ubuntu) or `yum install expect` (RHEL/CentOS). Pre-installed on macOS. |
| gog installed at wrong path | Architecture mismatch or mixed Homebrew install | Run `uname -m` to confirm architecture. Apple Silicon should use `/opt/homebrew/bin/gog`, Intel Mac should use `/usr/local/bin/gog`. Reinstall if path is unexpected. |
| OAuth token expired | Tokens expire after a period of inactivity | Re-authorize: `gog auth add <email> --client <name> --services=gmail,calendar,drive --manual`. |
| Browser does not open for OAuth | Headless environment, SSH session, or remote operator console | Use `--manual` flag. gog prints a URL to visit in any browser. After authorizing, copy the redirect URL from the browser's address bar. |
| Linux: Homebrew not available | Different package manager | Install via npm: `npm install -g @drizzle-team/gog`. |
| Token refresh fails silently | Corrupted token store | Delete `~/Library/Application Support/gogcli/keyring/` (macOS) or `~/.config/gogcli/keyring/` (Linux) and re-authenticate. |
| Backup folder already exists | Previous deployment or manual creation | Use the existing folder. Run `gog drive ls --plain \| grep -i backup` to find the folder ID. |
| `.env` has literal `$(which gog)` text | The echo command used single quotes instead of double quotes | Remove the bad line and re-run with double quotes: `echo "GOG_CLI_PATH=$(which gog)" >> ${WORKSPACE}/.env` |
| Cron jobs cannot find gog | PATH not set in cron environment | Source `${WORKSPACE}/.env` at the top of cron scripts to get `GOG_CLI_PATH`, `GOG_ACCOUNT`, and `GOG_CLIENT`. |
| Expect script has literal `\$argv` | Heredoc used single-quoted delimiter (`'EOF'`) which preserves backslashes literally | Use unquoted heredoc (`EOF`) with `\$` escaping for Tcl variables only. Shell variables get substituted, Tcl variables survive as `$var`. |
| Credentials JSON has literal backslash-quotes | Single-quoted heredoc preserved `\"` as literal two-character sequences | Use unquoted heredoc so shell expansion produces clean JSON with normal `"` characters. |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Detect architecture and existing install | Execute silently | Execute silently | Confirm each command |
| 1.2 Check existing backup folder | Execute silently | Execute silently | Confirm |
| 1.3 Check for expect | Execute silently | Execute silently | Confirm |
| 2.1 Choose services | Always ask client | Always ask client | Always ask client |
| 2.2 Install gog CLI | Execute silently | Confirm before install | Confirm |
| 2.3 Path A: default OAuth | Execute, fall through to Path B if blocked | Confirm before starting | Confirm |
| 2.4 Path B: custom OAuth setup | Wait for user credentials | Wait for user credentials | Wait for user credentials |
| 2.5 Path B: complete OAuth flow | Wait for user auth code | Wait for user auth code | Wait for user auth code |
| 2.6 Verify auth | Execute silently | Execute silently | Confirm |
| 2.7 Save gog defaults | Execute silently | Confirm before writing | Confirm |
| 3.1-3.3 Verify services | Execute silently | Execute silently | Confirm each |
| 4.1 Check existing backup folder | Execute silently | Execute silently | Confirm |
| 4.2 Create backup folder | Execute silently | Confirm before creating | Confirm |
| 4.3 Save folder ID | Execute silently | Confirm | Confirm |
| 5.1-5.2 Finalize config | Execute silently | Execute silently | Confirm each |
| 5.3 Update client profile | Execute silently | Execute silently | Show changes, confirm |

## Known Issues and Lessons Learned

These are hard-won lessons from real deployments. Read before attempting this skill.

### Remote OAuth Flow State Mismatch

**Problem:** `gog auth add --remote --step 1` / `--step 2` fails with "state mismatch" error. The `--remote` flag generates a state token in step 1, but by step 2 the state context is lost (each invocation generates a new state).

**Solution:** Use the `expect` workaround in Step 2.5. The expect script starts a single gog process, captures its state parameter from the OAuth URL it prints, and feeds the redirect URL back to the same process. This keeps the state consistent within a single process lifetime.

**Alternative solution:** Exchange the authorization code directly with Google's token endpoint:
```bash
curl -s -X POST https://oauth2.googleapis.com/token \
  -d "code=${AUTH_CODE}" \
  -d "client_id=${GCP_CLIENT_ID}" \
  -d "client_secret=${GCP_CLIENT_SECRET}" \
  -d "redirect_uri=http://localhost:1/" \
  -d "grant_type=authorization_code"
```
Then import the resulting tokens with `gog auth tokens import`.

### macOS Keychain Prompt

**Problem:** `gog auth tokens import` triggers a macOS Keychain password dialog on the physical screen. In a remote PTY session, this dialog is invisible to the operator and blocks the import.

**Solution:** Run `gog auth keyring file` BEFORE importing tokens. This tells gog to store credentials in a file instead of the macOS Keychain, avoiding the GUI dialog entirely. This is essential for fully automated remote deploys.

### gog Search JSON Format

When using `gog gmail search`, the output format depends on the `--plain` flag:
- With `--plain`: TSV output (one row per result, tab-separated fields)
- Without `--plain`: JSON with `{ "threads": [...] }` structure (NOT a flat array)

Each thread object contains: `{ id, date, from, subject, labels, messageCount }`.

### File-Based Credential Storage

For fully automated deploys, always configure file-based credential storage before any auth operations:
```
gog auth keyring file
```

This prevents any GUI password prompts that would block a remote PTY session.

## Dependencies

- **Depends on:** `openclaw/install.md` (OpenClaw must be installed and running)
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
