# Deploy Beehiiv + HubSpot Sync

## Purpose

Connect Beehiiv (newsletter platform) and HubSpot (sales CRM) for data sync and analysis. Caches data locally in SQLite. Feeds into the Business Advisory Council for cross-platform analysis.

**When to use:** When the user has Beehiiv and/or HubSpot accounts they want integrated.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "echo $BEEHIIV_API_KEY | head -c 8"
cmd --session <id> --command "echo $HUBSPOT_ACCESS_TOKEN | head -c 8"
```
- Does the client use Beehiiv? HubSpot? Both? Neither?
- Deploy only the integrations the client actually uses.

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Alternative | When to Adapt |
|-----------|---------|-------------|---------------|
| Newsletter platform | Beehiiv | ConvertKit, Mailchimp, Substack, none | Client uses a different newsletter platform |
| Sales CRM | HubSpot | Salesforce, Pipedrive, Close, none | Client uses a different CRM |
| Sync frequency | Every 4 hours | More/less frequent, manual only | Client preference and API rate limits |
| Data destination | SQLite cache + advisory council feed | SQLite only, or direct API queries | Client doesn't run the advisory council |
| Deploy scope | Both Beehiiv + HubSpot | One or the other | Client only uses one platform |

## Prerequisites

- `deploy-messaging-setup.md` completed
- For Beehiiv: `BEEHIIV_API_KEY` and `BEEHIIV_PUBLICATION_ID` in `.env`
- For HubSpot: `HUBSPOT_ACCESS_TOKEN` in `.env`

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
| Database | `~/clawd/data/beehiiv-sync.db` |
| Schedule | Every 4 hours |

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
| Database | `~/clawd/data/hubspot-sync.db` |
| Schedule | Every 4 hours |

### How Data Flows

1. Sync jobs pull data from Beehiiv and HubSpot APIs every 4 hours
2. Data is cached locally in SQLite databases
3. The Business Advisory Council reads from these databases during nightly analysis
4. Cross-platform insights (e.g., newsletter subscribers who became CRM deals) emerge from the advisory council's synthesis

## Steps

### 1. Configure Beehiiv Credentials

```
cmd --session <id> --command "grep BEEHIIV_API_KEY ~/clawd/.env"
cmd --session <id> --command "grep BEEHIIV_PUBLICATION_ID ~/clawd/.env"
```

If missing:

```
cmd --session <id> --command "echo 'BEEHIIV_API_KEY=<your-key>' >> ~/clawd/.env"
cmd --session <id> --command "echo 'BEEHIIV_PUBLICATION_ID=<your-pub-id>' >> ~/clawd/.env"
```

### 2. Configure HubSpot Credentials

```
cmd --session <id> --command "grep HUBSPOT_ACCESS_TOKEN ~/clawd/.env"
```

If missing:

```
cmd --session <id> --command "echo 'HUBSPOT_ACCESS_TOKEN=<your-token>' >> ~/clawd/.env"
```

### 3. Create Data Directory

```
cmd --session <id> --command "mkdir -p ~/clawd/data"
```

### 4. Install Skills and Sync Scripts

```
cmd --session <id> --command "cp -r /path/to/beehiiv ~/clawd/skills/beehiiv"
cmd --session <id> --command "cp -r /path/to/hubspot ~/clawd/skills/hubspot"
```

Install sync script dependencies:

```
cmd --session <id> --command "cd ~/clawd/tools/business-meta-analysis && npm install"
```

### 5. Run Initial Beehiiv Sync

```
cmd --session <id> --command "node ~/clawd/tools/business-meta-analysis/sync/beehiiv-sync.cjs"
```

Expected: Pulls publication stats and post-level data into `~/clawd/data/beehiiv-sync.db`.

### 6. Run Initial HubSpot Sync

```
cmd --session <id> --command "node ~/clawd/tools/business-meta-analysis/sync/hubspot-sync.cjs"
```

Expected: Pulls deals and contacts into `~/clawd/data/hubspot-sync.db`.

### 7. Set Up 4-Hour Sync Cron Jobs

```
cmd --session <id> --command "openclaw cron add --name 'Beehiiv Sync' --schedule '0 */4 * * *' --tz 'America/Los_Angeles' --command 'node ~/clawd/tools/business-meta-analysis/sync/beehiiv-sync.cjs'"
cmd --session <id> --command "openclaw cron add --name 'HubSpot Sync' --schedule '0 */4 * * *' --tz 'America/Los_Angeles' --command 'node ~/clawd/tools/business-meta-analysis/sync/hubspot-sync.cjs'"
```

### 8. Verify Data Populates Advisory Council

If the advisory council is already deployed, run a dry run to confirm it picks up the new data sources.

```
cmd --session <id> --command "cd ~/clawd/tools/business-meta-analysis && node scripts/nightly-business-meta-analysis.js --dry-run"
```

Check that the RevenueGuardian, GrowthStrategist, and CFO personas now have Beehiiv and HubSpot data in their inputs.

## Verification

Check sync status for both platforms:

```
node ~/clawd/tools/business-meta-analysis/sync/beehiiv-sync.cjs --status
node ~/clawd/tools/business-meta-analysis/sync/hubspot-sync.cjs --status
```

Expected output for each:
- Last sync timestamp
- Record counts (subscribers, posts for Beehiiv; deals, contacts for HubSpot)
- No error messages

Verify databases exist and have data:

```
cmd --session <id> --command "ls -la ~/clawd/data/beehiiv-sync.db ~/clawd/data/hubspot-sync.db"
```

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Beehiiv auth failure | API key invalid or revoked | Generate a new API key in Beehiiv dashboard > Settings > API. Update `.env`. |
| HubSpot auth failure | Access token expired or wrong scopes | Generate a new token in HubSpot > Settings > Integrations > Private Apps. Ensure it has `crm.objects.contacts.read` and `crm.objects.deals.read` scopes. |
| No Beehiiv data | Publication ID doesn't match | Verify `BEEHIIV_PUBLICATION_ID` matches the publication in your Beehiiv dashboard. |
| HubSpot sync slow | Large contact/deal database | The sync uses pagination. First sync may take several minutes for large CRMs. Subsequent syncs are incremental. |
| Sync cron not running | Cron job misconfigured | Check `openclaw cron list` for the sync jobs. Verify they appear and show recent run times. |
| Stale data | Cron jobs stopped or failing silently | Check cron-log for errors. Most common: expired tokens. Refresh credentials and re-run. |
| Advisory council doesn't see data | Source not registered | Check `source_catalog` table in `business-meta-analysis.db`. Run the advisory council's source registration if Beehiiv/HubSpot are missing. |

## Dependencies

- **Requires:** `deploy-messaging-setup.md`
- **Feeds into:** `deploy-advisory-council.md` (Beehiiv and HubSpot data used by RevenueGuardian, GrowthStrategist, CFO personas)
- **Independent of:** `deploy-asana.md`, `deploy-social-tracking.md` (but all enhance the advisory council together)
