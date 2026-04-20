# Deploy Business Advisory Council

## Purpose

Install a nightly business analysis system with 8 parallel independent AI expert personas. Aggregates data from 14 sources, runs expert analysis in parallel (so they can't influence each other), synthesizes findings into ranked recommendations delivered to Telegram.

**When to use:** After multiple data sources are deployed (CRM, social tracking, integrations). The more data sources connected, the more valuable the analysis.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "ls ${WORKSPACE}/tools/business-meta-analysis/ 2>/dev/null"
cmd --session <id> --command "openclaw config get models.primary"
```
- How many data sources are available? The council improves with more data, but works with as few as 3-4 sources.
- Which expert personas are relevant? Not every client needs all 8.
- What AI provider is available? (Anthropic preferred, but falls back)

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Alternative | When to Adapt |
|-----------|---------|-------------|---------------|
| Expert personas | All 8 (RevenueGuardian, GrowthStrategist, SkepticalOperator, TeamDynamicsArchitect, AutomationScout, CFO, ContentStrategist, MarketAnalyst) | Subset of relevant personas | Client doesn't need all domains. Solo creator might skip TeamDynamicsArchitect. Non-content business might skip ContentStrategist. |
| Data sources | 14 sources (YouTube, IG, X, CRM, email, Fathom, cron, Slack, Asana, HubSpot, Beehiiv, financials, etc.) | Only connected sources | Council gracefully handles missing sources. Deploy with what's available. |
| AI provider | Anthropic (Opus/Sonnet) | OpenAI, Google Gemini | Client's available API keys |
| Run schedule | Nightly 4:30am PST | Different time, weekly instead of daily, manual only | Client preference or cost concerns |
| Notification channel | Telegram Meta-Analysis topic | Discord channel, Slack channel, email digest | Client's messaging platform |
| Financial confidentiality | Directional only (no specific $) | Include specifics (DM delivery) | Client's comfort level with financial data in digests |
| Sync schedules | Slack 3h, Asana 4h, HubSpot 4h | Adjust frequency | Cost, API limits, or data freshness needs |
| Deeper dives | "Tell me more about #N" in Telegram | CLI-only, different channel | Client's interaction preference |

## Prerequisites

- `deploy-messaging-setup.md` completed (Telegram Meta-Analysis topic)
- `deploy-security-safety.md` completed
- At least some data sources deployed (more is better):
  - `deploy-personal-crm.md` -- CRM contacts, email activity
  - `deploy-social-tracking.md` -- YouTube, Instagram, X/Twitter analytics
  - `deploy-newsletter-crm.md` -- Beehiiv + HubSpot data (optional)
  - `deploy-asana.md` -- Task status (optional)
- AI API access (Anthropic preferred, falls back to OpenAI/Google)

## What Gets Installed

### 8 Expert Personas (Run in Parallel)

Each expert sees only their relevant data slices. They run independently so no persona can influence another's analysis.

| Persona | Domain | Data Sources |
|---------|--------|-------------|
| RevenueGuardian | Revenue protection | Financials, HubSpot deals, YouTube earnings |
| GrowthStrategist | Growth opportunities | YouTube, Instagram, X/Twitter, newsletter |
| SkepticalOperator | Operational reality check | Cron logs, error rates, system health |
| TeamDynamicsArchitect | Team and workflow | Slack, Asana, meeting patterns |
| AutomationScout | Automation opportunities | Cron reliability, manual task patterns |
| CFO | Financial health | Financials (directional only, not specific $), burn rate, AR/AP |
| ContentStrategist | Content performance | YouTube per-video, Instagram, X/Twitter analytics |
| MarketAnalyst | Market signals | X/Twitter trends, competitor data, industry news |

### Data Collectors (14 Sources)

YouTube, Instagram per-post, Instagram account growth, X/Twitter per-post, X/Twitter trends, CRM contacts, email activity, Fathom meetings, cron reliability, Slack messages, Asana tasks, HubSpot pipeline + contacts, Beehiiv newsletter stats, Financials

### Architecture

- Each expert only sees data from their tagged sources plus a cross-domain brief
- All 8 run in parallel (independent -- can't influence each other)
- Synthesizer merges findings, eliminates duplicates, ranks by priority
- Numbered digest delivered to Telegram
- "Tell me more about #3" triggers deeper dives
- Feedback loop: approve/reject recommendations, system learns preferences

### Database (`~/clawd/data/business-meta-analysis.db`)

| Table | Purpose |
|-------|---------|
| source_catalog | Registered data sources with tags and freshness |
| signal_runs | Analysis run metadata (timestamp, duration, model used) |
| analysis_artifacts | Raw expert outputs per run |
| recommendations | Synthesized, ranked recommendations |
| feedback | Approve/reject history for learning |
| deeper_dive_requests | Follow-up analysis triggers |
| weight_policies | Per-persona and per-source weighting rules |

### Sync Jobs

| Sync | Schedule | What |
|------|----------|------|
| Slack Sync | Every 3 hours | Slack messages |
| Asana Sync | Every 4 hours | Asana tasks |
| HubSpot Sync | Every 4 hours | Deals + contacts |
| Beehiiv Sync | (part of analysis run) | Newsletter stats |

### Cron

Nightly Business Meta Analysis -- Daily 4:30am PST

### Telegram Integration

- Meta-Analysis topic (ID: 1827) for numbered digests and deeper dives

## Steps

### 1. Create Business Meta-Analysis Directory

```
cmd --session <id> --command "mkdir -p ~/clawd/tools/business-meta-analysis/sync ~/clawd/tools/business-meta-analysis/scripts ~/clawd/tools/business-meta-analysis/personas"
```

### 2. Install Dependencies

```
cmd --session <id> --command "cd ~/clawd/tools/business-meta-analysis && npm install"
```

### 3. Configure Data Source Connections

Set up connection details for each available data source in the environment.

```
cmd --session <id> --command "openclaw config set advisory-council.anthropic-key '<ANTHROPIC_API_KEY>'"
cmd --session <id> --command "openclaw config set advisory-council.fallback-provider openai"
```

For each optional source, configure as available:

```
cmd --session <id> --command "openclaw config set advisory-council.hubspot-token '<HUBSPOT_TOKEN>'"
cmd --session <id> --command "openclaw config set advisory-council.beehiiv-api-key '<BEEHIIV_API_KEY>'"
cmd --session <id> --command "openclaw config set advisory-council.asana-token '<ASANA_TOKEN>'"
```

### 4. Run Sync Jobs to Populate Initial Data

Run each sync manually to establish baseline data before the first analysis.

```
cmd --session <id> --command "cd ~/clawd/tools/business-meta-analysis && node sync/slack-sync.cjs"
cmd --session <id> --command "cd ~/clawd/tools/business-meta-analysis && node sync/asana-sync.cjs"
cmd --session <id> --command "cd ~/clawd/tools/business-meta-analysis && node sync/hubspot-sync.cjs"
```

### 5. Run a Manual Analysis (Dry Run)

Test the full pipeline without delivering to Telegram.

```
cmd --session <id> --command "cd ~/clawd/tools/business-meta-analysis && node scripts/nightly-business-meta-analysis.js --dry-run"
```

Review the output to confirm all 8 personas produced analysis and the synthesizer ranked recommendations correctly.

### 6. Set Up Nightly Cron at 4:30am PST

```
cmd --session <id> --command "openclaw cron add --name 'Nightly Business Meta Analysis' --schedule '30 4 * * *' --tz 'America/Los_Angeles' --command 'cd ~/clawd/tools/business-meta-analysis && node scripts/nightly-business-meta-analysis.js'"
```

### 7. Set Up Sync Cron Jobs

```
cmd --session <id> --command "openclaw cron add --name 'Slack Sync' --schedule '0 */3 * * *' --tz 'America/Los_Angeles' --command 'cd ~/clawd/tools/business-meta-analysis && node sync/slack-sync.cjs'"
cmd --session <id> --command "openclaw cron add --name 'Asana Sync' --schedule '0 */4 * * *' --tz 'America/Los_Angeles' --command 'cd ~/clawd/tools/business-meta-analysis && node sync/asana-sync.cjs'"
cmd --session <id> --command "openclaw cron add --name 'HubSpot Sync' --schedule '0 */4 * * *' --tz 'America/Los_Angeles' --command 'cd ~/clawd/tools/business-meta-analysis && node sync/hubspot-sync.cjs'"
```

### 8. Configure Telegram Meta-Analysis Topic

```
cmd --session <id> --command "openclaw config set advisory-council.telegram-topic 1827"
```

### 9. Test Deeper Dive

Test that deeper-dive requests work by requesting expansion on a recommendation from the dry run.

```
cmd --session <id> --command "cd ~/clawd/tools/business-meta-analysis && node scripts/council-deeper-dive.js --council platform --number 1"
```

## Verification

Run these commands to confirm the advisory council is operational:

```
cmd --session <id> --command "cd ~/clawd/tools/business-meta-analysis && node sync/hubspot-sync.cjs --status"
cmd --session <id> --command "cd ~/clawd/tools/business-meta-analysis && node scripts/nightly-business-meta-analysis.js --dry-run --json"
```

Expected output:
- Sync status should show last sync time and record counts for each source
- Dry run should produce JSON with 8 persona outputs and a synthesized ranked list
- Check Telegram Meta-Analysis topic for numbered digest delivery

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Empty analysis | Not enough data sources connected | Deploy more data systems first. The council produces better results with more data. Even 3-4 sources is enough to start. |
| API errors during analysis | Model provider keys missing or invalid | Check `ANTHROPIC_API_KEY` first (preferred). Falls back to `OPENAI_API_KEY` then `GEMINI_API_KEY`. |
| Sync failures | Individual sync job errors | Check sync job logs in cron-log. Most common: expired tokens (HubSpot, Asana) or rate limits (Slack). |
| Confidentiality violation | Financial data too specific in digest | The CFO persona uses directional language only (e.g., "revenue trending up" not "$X,XXX"). If specific amounts appear, check the CFO persona prompt. |
| Deeper dive returns nothing | Referenced recommendation number doesn't exist | Use the exact number from the most recent digest. Numbers reset each run. |
| Personas influencing each other | Architecture bug | All 8 must run in parallel with isolated contexts. Check that the runner is using `Promise.all()` and not sequential execution. |

## Dependencies

- **Requires:** `deploy-messaging-setup.md`, `deploy-security-safety.md`
- **Enhanced by:** `deploy-personal-crm.md`, `deploy-social-tracking.md`, `deploy-newsletter-crm.md`, `deploy-asana.md`, `deploy-fathom-pipeline.md`
