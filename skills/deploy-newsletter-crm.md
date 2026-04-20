---
name: "Deploy Beehiiv + HubSpot Sync"
category: "integration"
subcategory: "crm"
third_party_service: "Beehiiv + HubSpot"
auth_type: "api-key"
openclaw_version_required: "2026.4.15+"
version_last_verified: "2026-04-20"
last_checked: "2026-04-20"
maintenance_status: "active"
model_env_var: "n/a"
clawhub_alternative: "beehiiv"
cost_tier: "api-usage"
privacy_tier: "cloud"
requires:
  - "Beehiiv API key"
  - "HubSpot private app access token"
  - "SQLite"
  - "openclaw cron (2026.4.15+)"
docs_urls:
  - "https://developers.beehiiv.com/"
  - "https://developers.hubspot.com/docs/api/overview"
---

# Deploy Beehiiv + HubSpot Sync

## Compatibility

- **OpenClaw Version**: 2026.4.15+ (compat header rewritten 2026-04-20 — live-verified on 4C)
- **Status**: WORKING — previous "BLOCKED" was stale. On 2026.4.15: `openclaw cron` (add/list/rm — NOT `remove`) is available per `audit/_baseline.md` §3; `openclaw status` does NOT exist (never did) — use `openclaw health` (§4C.7)
- **Scheduling**: `openclaw cron add --name beehiiv-sync --cron "0 */4 * * *"` and `--name hubspot-sync --cron "30 */4 * * *"` (stagger to avoid simultaneous load)
- **Config**: use `openclaw config set` for scalar API endpoints/limits; edit JSON for arrays
- **Model**: agent default (Codex-first); do not hardcode
- **ClawHub alternatives**: `beehiiv` (managed-OAuth), `beehiiv-integration`, `newsletter-digest` per §B

## Purpose

Connect Beehiiv (newsletter platform) and HubSpot (sales CRM) for data sync and analysis. Pulls subscriber metrics, post-level analytics, deal pipelines, and contact records into local SQLite databases on a recurring schedule. These databases feed into the Business Advisory Council for cross-platform analysis (e.g., identifying newsletter subscribers who became CRM deals).

**When to use:** When the client has Beehiiv and/or HubSpot accounts they want integrated into the OpenClaw ecosystem.

**What this skill does:**
1. Configures API credentials for Beehiiv and/or HubSpot
2. Creates the data directory and installs sync scripts with dependencies
3. Runs initial data syncs to populate local SQLite caches
4. Schedules recurring 4-hour sync cron jobs
5. Verifies data flows into the advisory council (if deployed)

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

Resolve these before executing. Sources are listed for each.

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${BEEHIIV_API_KEY}` | Client provides (Beehiiv dashboard > Settings > API) | `bh_live_abc123...` |
| `${BEEHIIV_PUBLICATION_ID}` | Client provides (Beehiiv dashboard > Settings > Publication) | `pub_abc123...` |
| `${HUBSPOT_ACCESS_TOKEN}` | Client provides (HubSpot > Settings > Integrations > Private Apps) | `pat-na1-abc123...` |
| `${TIMEZONE}` | Client profile: timezone | `America/Los_Angeles` |
| `${SYNC_SCHEDULE}` | Client preference, default `0 */4 * * *` | `0 */4 * * *` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**

Determine which platforms the client uses. Ask the client directly or check the client profile. Only deploy integrations the client actually uses.

Check if Beehiiv credentials already exist:

**Remote:**
```
grep BEEHIIV_API_KEY ${WORKSPACE}/.env 2>/dev/null && echo 'BEEHIIV_CONFIGURED' || echo 'BEEHIIV_NOT_CONFIGURED'
```

Check if HubSpot credentials already exist:

**Remote:**
```
grep HUBSPOT_ACCESS_TOKEN ${WORKSPACE}/.env 2>/dev/null && echo 'HUBSPOT_CONFIGURED' || echo 'HUBSPOT_NOT_CONFIGURED'
```

Check if sync databases already exist (indicates a previous deployment):

**Remote:**
```
ls -la ${WORKSPACE}/data/beehiiv-sync.db ${WORKSPACE}/data/hubspot-sync.db 2>/dev/null || echo 'NO_EXISTING_DBS'
```

Check if sync cron jobs are already scheduled:

**Remote:**
```
openclaw cron list 2>/dev/null | grep -i -E 'beehiiv|hubspot' || echo 'NO_EXISTING_CRON'
```

**Decision points from pre-flight:**
- Does the client use Beehiiv? HubSpot? Both? Deploy only the integrations they actually use.
- Are credentials already configured? If so, verify they work before proceeding.
- Are sync databases present? If so, this may be a redeployment -- check data freshness.
- Are cron jobs already running? If so, check if the schedule matches the desired frequency.

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Newsletter platform | Beehiiv | Client uses ConvertKit, Mailchimp, Substack, or no newsletter platform |
| Sales CRM | HubSpot | Client uses Salesforce, Pipedrive, Close, or no CRM |
| Sync frequency | Every 4 hours (`0 */4 * * *`) | Client wants more/less frequent syncs, or manual-only |
| Data destination | SQLite cache + advisory council feed | Client doesn't run the advisory council (SQLite only) |
| Deploy scope | Both Beehiiv + HubSpot | Client only uses one of the two platforms |
| Sync timezone | `${TIMEZONE}` from client profile | Client wants syncs aligned to a different timezone |

## Prerequisites

- OpenClaw installed and running (`openclaw/install.md`)
- `deploy-messaging-setup.md` completed (for notification of sync failures)
- Node.js available on the client machine (installed with OpenClaw)
- For Beehiiv: API key and publication ID from the client
- For HubSpot: Private app access token with `crm.objects.contacts.read` and `crm.objects.deals.read` scopes

## What Gets Installed

### Beehiiv Skill (`skills/beehiiv/`)

| Metric | Description |
|--------|-------------|
| Subscriber count | Total active subscribers |
| Growth rate | New subscribers over time |
| Churn | Unsubscribe rate |
| Per-post open rates | Open rate for each newsletter edition |
| Per-post click rates | Click-through rate for each edition |
| Subscriber segments | Audience segments and their sizes |

### Beehiiv Sync (`tools/business-meta-analysis/sync/beehiiv-sync.cjs`)

| Data | Details |
|------|---------|
| Publication stats | Subscriber counts, open/click rates, churn |
| Post-level data | Subject lines, publish dates, open/click rates, recipients, unsubscribes |
| Database | `${WORKSPACE}/data/beehiiv-sync.db` |
| Schedule | Every 4 hours (configurable) |

### HubSpot Skill (`skills/hubspot/`)

| Capability | Description |
|------------|-------------|
| Contact management | Create, read, update contacts via REST API |
| Company management | Company records and associations |
| Deal management | Pipeline stages, values, owners |
| Content management | Marketing content via HubSpot CMS |

### HubSpot Sync (`tools/business-meta-analysis/sync/hubspot-sync.cjs`)

| Data | Details |
|------|---------|
| Deals | Stage, value, owner, type, description (paginated) |
| Contacts | Email, name, company, job title, phone, lifecycle stage |
| Database | `${WORKSPACE}/data/hubspot-sync.db` |
| Schedule | Every 4 hours (configurable) |

### How Data Flows

1. Sync jobs pull data from Beehiiv and HubSpot APIs every 4 hours
2. Data is cached locally in SQLite databases
3. The Business Advisory Council reads from these databases during nightly analysis
4. Cross-platform insights (e.g., newsletter subscribers who became CRM deals) emerge from the advisory council's synthesis

## Steps

### Phase 1: Credentials `[HUMAN_INPUT]`

API keys must come from the client. The operator cannot generate these.

#### 1.1 Configure Beehiiv Credentials `[HUMAN_INPUT]`

**Skip this step if the client does not use Beehiiv.**

Ask the client for their Beehiiv API key and publication ID. They can find these at:
- API key: Beehiiv dashboard > Settings > API
- Publication ID: Beehiiv dashboard > Settings > Publication (the `pub_` prefixed ID)

Once the client provides the actual values, append them to the environment file.

**Remote:**
```
printf 'BEEHIIV_API_KEY=%s\n' '<actual-api-key-from-client>' >> ${WORKSPACE}/.env
printf 'BEEHIIV_PUBLICATION_ID=%s\n' '<actual-publication-id-from-client>' >> ${WORKSPACE}/.env
```

> **Important:** Replace `<actual-api-key-from-client>` and `<actual-publication-id-from-client>` with the real values the client provides. Do NOT send the placeholder strings -- they will be written literally to the file and the sync will fail.

Expected: The `.env` file contains `BEEHIIV_API_KEY=bh_live_...` and `BEEHIIV_PUBLICATION_ID=pub_...` with real values.

If this fails: Check file permissions on `${WORKSPACE}/.env`. Ensure the workspace directory exists.

If already exists: Check that the existing values are correct and current. Beehiiv keys can be revoked. If updating, remove the old lines first: `sed -i.bak '/^BEEHIIV_/d' ${WORKSPACE}/.env` then re-add.

#### 1.2 Configure HubSpot Credentials `[HUMAN_INPUT]`

**Skip this step if the client does not use HubSpot.**

Ask the client for their HubSpot private app access token. They can find it at:
- HubSpot > Settings > Integrations > Private Apps
- The token must have scopes: `crm.objects.contacts.read` and `crm.objects.deals.read`

Once the client provides the actual value, append it to the environment file.

**Remote:**
```
printf 'HUBSPOT_ACCESS_TOKEN=%s\n' '<actual-access-token-from-client>' >> ${WORKSPACE}/.env
```

> **Important:** Replace `<actual-access-token-from-client>` with the real token the client provides. Do NOT send the placeholder string.

Expected: The `.env` file contains `HUBSPOT_ACCESS_TOKEN=pat-na1-...` with the real token value.

If this fails: Check file permissions on `${WORKSPACE}/.env`.

If already exists: Check that the existing token is valid and has the required scopes. HubSpot tokens can expire. If updating, remove the old line first: `sed -i.bak '/^HUBSPOT_ACCESS_TOKEN/d' ${WORKSPACE}/.env` then re-add.

#### 1.3 Verify Credentials Were Written Correctly `[AUTO]`

Confirm the credentials are present and are real values (not placeholder text).

**Remote:**
```
grep -E '^BEEHIIV_API_KEY=.{10,}' ${WORKSPACE}/.env 2>/dev/null && echo 'BEEHIIV_KEY_OK' || echo 'BEEHIIV_KEY_MISSING_OR_SHORT'
grep -E '^BEEHIIV_PUBLICATION_ID=pub_' ${WORKSPACE}/.env 2>/dev/null && echo 'BEEHIIV_PUB_OK' || echo 'BEEHIIV_PUB_MISSING'
grep -E '^HUBSPOT_ACCESS_TOKEN=.{10,}' ${WORKSPACE}/.env 2>/dev/null && echo 'HUBSPOT_OK' || echo 'HUBSPOT_MISSING_OR_SHORT'
```

Expected: `*_OK` for each platform the client uses. `*_MISSING` is acceptable for platforms being skipped.

If this fails: Re-check that step 1.1 / 1.2 used actual values, not placeholder strings like `<your-key>`. If the `.env` contains literal angle-bracket placeholders, delete those lines and redo the credential steps with real values.

### Phase 2: Data Directory and Dependencies `[AUTO]`

#### 2.1 Create Data Directory `[AUTO]`

Ensure the data directory exists for the sync databases.

**Remote:**
```
mkdir -p ${WORKSPACE}/data
```

Expected: Directory exists at `${WORKSPACE}/data/`.

If already exists: `mkdir -p` is idempotent. No action needed.

#### 2.2 Install Skill Definitions `[AUTO]`

Copy the Beehiiv and HubSpot skill directories into the workspace. These define the capabilities the agent has for querying each platform.

**Remote:**
```
ls ${WORKSPACE}/skills/beehiiv/ 2>/dev/null && echo 'BEEHIIV_SKILL_EXISTS' || echo 'BEEHIIV_SKILL_MISSING'
ls ${WORKSPACE}/skills/hubspot/ 2>/dev/null && echo 'HUBSPOT_SKILL_EXISTS' || echo 'HUBSPOT_SKILL_MISSING'
```

If skills are missing, copy them from the source. The source location depends on how OpenClaw was installed -- check for the skill templates in the OpenClaw installation or workspace template directory.

**Remote (if missing):**
```
cp -r ${WORKSPACE}/templates/skills/beehiiv ${WORKSPACE}/skills/beehiiv
cp -r ${WORKSPACE}/templates/skills/hubspot ${WORKSPACE}/skills/hubspot
```

Expected: `${WORKSPACE}/skills/beehiiv/` and `${WORKSPACE}/skills/hubspot/` exist with skill definition files.

If this fails: The skill templates may be in a different location. Check `ls ${WORKSPACE}/templates/` or search for them: `find ${WORKSPACE} -name 'beehiiv' -type d 2>/dev/null`. If templates are not bundled, the skill files may need to be created manually using the patterns from the OpenClaw documentation.

If already exists: Compare content to see if an update is needed. If identical, skip.

#### 2.3 Install Sync Script Dependencies `[AUTO]`

Install Node.js dependencies for the sync scripts.

**Remote:**
```
cd ${WORKSPACE}/tools/business-meta-analysis && npm install
```

Expected: `node_modules/` created inside `${WORKSPACE}/tools/business-meta-analysis/` with no errors.

If this fails: Check that Node.js is available (`node --version`). Check that `package.json` exists in the directory. If the directory doesn't exist, the business-meta-analysis tooling may not be deployed yet -- check if `deploy-advisory-council.md` needs to be run first.

If already exists: `npm install` is idempotent. Re-running it updates dependencies if `package.json` changed.

### Phase 3: Initial Sync `[GUIDED]`

Run the first data sync for each platform. The initial sync may take several minutes for large datasets (HubSpot with thousands of contacts/deals).

#### 3.1 Run Initial Beehiiv Sync `[GUIDED]`

**Skip if the client does not use Beehiiv.**

Pull all publication stats and post-level data into the local SQLite database.

**Remote:**
```
cd ${WORKSPACE} && node tools/business-meta-analysis/sync/beehiiv-sync.cjs
```

Expected: Script completes without errors. Creates `${WORKSPACE}/data/beehiiv-sync.db` with publication stats and per-post metrics (open rates, click rates, subscriber counts).

If this fails:
- **Auth error (401/403):** The `BEEHIIV_API_KEY` is invalid or revoked. Ask the client to generate a new key.
- **Publication not found (404):** The `BEEHIIV_PUBLICATION_ID` doesn't match. Ask the client to verify the correct publication ID from their dashboard.
- **Rate limit (429):** Wait a few minutes and retry. Beehiiv rate limits are generous but can be hit during large initial syncs.
- **Module not found:** Dependencies weren't installed. Re-run step 2.3.

#### 3.2 Run Initial HubSpot Sync `[GUIDED]`

**Skip if the client does not use HubSpot.**

Pull all deals and contacts into the local SQLite database. The sync uses pagination, so large CRMs may take several minutes on the first run. Subsequent syncs are incremental.

**Remote:**
```
cd ${WORKSPACE} && node tools/business-meta-analysis/sync/hubspot-sync.cjs
```

Expected: Script completes without errors. Creates `${WORKSPACE}/data/hubspot-sync.db` with deals (stage, value, owner, type, description) and contacts (email, name, company, job title, phone, lifecycle stage).

If this fails:
- **Auth error (401/403):** The `HUBSPOT_ACCESS_TOKEN` is invalid, expired, or missing required scopes. Ask the client to generate a new private app token with `crm.objects.contacts.read` and `crm.objects.deals.read` scopes.
- **Rate limit (429):** HubSpot has a 100-requests-per-10-seconds limit. The sync script handles pagination, but very large datasets may hit this. Wait and retry.
- **Timeout:** For CRMs with 10,000+ records, the first sync can take 5+ minutes. Let it run.
- **Module not found:** Dependencies weren't installed. Re-run step 2.3.

#### 3.3 Verify Sync Databases Have Data `[AUTO]`

Confirm the databases were created and contain records.

**Remote:**
```
ls -la ${WORKSPACE}/data/beehiiv-sync.db ${WORKSPACE}/data/hubspot-sync.db 2>/dev/null
```

**Remote (check Beehiiv record count):**
```
sqlite3 ${WORKSPACE}/data/beehiiv-sync.db "SELECT COUNT(*) FROM posts;" 2>/dev/null || echo 'NO_BEEHIIV_DATA'
```

**Remote (check HubSpot record counts):**
```
sqlite3 ${WORKSPACE}/data/hubspot-sync.db "SELECT 'deals: ' || COUNT(*) FROM deals UNION ALL SELECT 'contacts: ' || COUNT(*) FROM contacts;" 2>/dev/null || echo 'NO_HUBSPOT_DATA'
```

Expected: Database files exist and contain records. Beehiiv should have posts; HubSpot should have deals and contacts.

If this fails: Re-run the sync for the failing platform (steps 3.1 or 3.2). Check the sync script output for error messages.

### Phase 4: Schedule Recurring Syncs `[AUTO]`

#### 4.1 Schedule Beehiiv Sync Cron `[AUTO]`

**Skip if the client does not use Beehiiv.**

Schedule the Beehiiv sync to run on the configured schedule (default: every 4 hours).

**Remote:**
```
openclaw cron add --name 'Beehiiv Sync' --schedule '${SYNC_SCHEDULE}' --tz '${TIMEZONE}' --command 'cd ${WORKSPACE} && node tools/business-meta-analysis/sync/beehiiv-sync.cjs'
```

Expected: Cron job registered and visible in `openclaw cron list`.

If this fails: Check that OpenClaw cron is functional: `openclaw cron list`. If the cron system isn't working, check that the OpenClaw gateway is running: `openclaw health` (or `openclaw gateway status`).

If already exists: Check if the existing cron schedule matches. If it does, skip. If different, remove the old job and add the new one: `openclaw cron rm --name 'Beehiiv Sync'` then re-add.

#### 4.2 Schedule HubSpot Sync Cron `[AUTO]`

**Skip if the client does not use HubSpot.**

Schedule the HubSpot sync to run on the configured schedule (default: every 4 hours).

**Remote:**
```
openclaw cron add --name 'HubSpot Sync' --schedule '${SYNC_SCHEDULE}' --tz '${TIMEZONE}' --command 'cd ${WORKSPACE} && node tools/business-meta-analysis/sync/hubspot-sync.cjs'
```

Expected: Cron job registered and visible in `openclaw cron list`.

If this fails: Same troubleshooting as step 4.1.

If already exists: Check if the existing cron schedule matches. If it does, skip. If different, remove and re-add.

#### 4.3 Verify Cron Jobs Are Registered `[AUTO]`

**Remote:**
```
openclaw cron list | grep -i -E 'beehiiv|hubspot'
```

Expected: One or both sync jobs listed with the correct schedule and timezone.

### Phase 5: Advisory Council Integration `[GUIDED]`

This phase only applies if `deploy-advisory-council.md` has already been deployed. If not, skip this phase -- the advisory council will automatically pick up the sync databases when it is deployed later.

#### 5.1 Verify Advisory Council Sees New Data Sources `[GUIDED]`

Run a dry run of the advisory council to confirm it detects the Beehiiv and HubSpot data.

**Remote:**
```
cd ${WORKSPACE}/tools/business-meta-analysis && node scripts/nightly-business-meta-analysis.js --dry-run
```

Expected: The dry run output shows that the RevenueGuardian, GrowthStrategist, and CFO personas now include Beehiiv and/or HubSpot data in their inputs.

If this fails: Check the `source_catalog` table in the advisory council's database. If Beehiiv/HubSpot sources are missing, they may need to be registered. Consult `deploy-advisory-council.md` for the source registration steps.

## Verification

Run these post-deployment checks to confirm everything is working.

Check sync status for both platforms:

**Remote:**
```
node ${WORKSPACE}/tools/business-meta-analysis/sync/beehiiv-sync.cjs --status
node ${WORKSPACE}/tools/business-meta-analysis/sync/hubspot-sync.cjs --status
```

Expected for each:
- Last sync timestamp (recent)
- Record counts (subscribers, posts for Beehiiv; deals, contacts for HubSpot)
- No error messages

Verify databases exist and have data:

**Remote:**
```
ls -la ${WORKSPACE}/data/beehiiv-sync.db ${WORKSPACE}/data/hubspot-sync.db 2>/dev/null
```

Expected: Both files present (for platforms that were deployed), non-zero size.

Verify cron jobs are scheduled:

**Remote:**
```
openclaw cron list | grep -i -E 'beehiiv|hubspot'
```

Expected: Sync jobs listed with correct schedule.

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Beehiiv auth failure | API key invalid or revoked | Generate a new API key in Beehiiv dashboard > Settings > API. Update `${WORKSPACE}/.env`. |
| HubSpot auth failure | Access token expired or wrong scopes | Generate a new token in HubSpot > Settings > Integrations > Private Apps. Ensure it has `crm.objects.contacts.read` and `crm.objects.deals.read` scopes. Update `${WORKSPACE}/.env`. |
| No Beehiiv data | Publication ID doesn't match | Verify `BEEHIIV_PUBLICATION_ID` matches the publication in the Beehiiv dashboard. |
| HubSpot sync slow | Large contact/deal database | The sync uses pagination. First sync may take several minutes for large CRMs. Subsequent syncs are incremental. |
| Sync cron not running | Cron job misconfigured | Check `openclaw cron list` for the sync jobs. Verify they appear and show recent run times. |
| Stale data | Cron jobs stopped or failing silently | Check cron-log for errors. Most common: expired tokens. Refresh credentials and re-run. |
| Advisory council doesn't see data | Source not registered | Check `source_catalog` table in `business-meta-analysis.db`. Run the advisory council's source registration if Beehiiv/HubSpot are missing. |
| `.env` contains `<your-key>` literal text | Operator sent placeholder instead of real value | Delete the placeholder lines (`sed -i.bak '/your-key/d' ${WORKSPACE}/.env`), get real values from client, redo credential steps. |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1-1.2 Configure credentials | **Pause** -- always requires client input `[HUMAN_INPUT]` | **Pause** -- always requires client input | **Pause** -- always requires client input |
| 1.3 Verify credentials | Execute silently | Execute silently | Show output, confirm |
| 2.1-2.3 Data dir + dependencies | Execute silently | Confirm before phase | Confirm each command |
| 3.1-3.2 Initial syncs | Execute silently | Confirm before each sync | Confirm each command |
| 3.3 Verify sync data | Execute silently | Execute silently | Show output, confirm |
| 4.1-4.3 Schedule cron jobs | Execute silently | Confirm before scheduling | Confirm each cron job |
| 5.1 Advisory council check | Execute silently | Confirm before running | Confirm, show output |

## Dependencies

- **Depends on:** `openclaw/install.md` (OpenClaw must be installed), `deploy-messaging-setup.md` (for failure notifications)
- **Feeds into:** `deploy-advisory-council.md` (Beehiiv and HubSpot data used by RevenueGuardian, GrowthStrategist, CFO personas)
- **Independent of:** `deploy-asana.md`, `deploy-social-tracking.md` (but all enhance the advisory council together)
