# Deploy Business Advisory Council

## Compatibility

- **OpenClaw Version**: 2026.4.15+ (compat header rewritten 2026-04-19 — live-verified on 4C)
- **Status**: WORKING — previous "BLOCKED" status was stale. On 2026.4.15: `openclaw cron` (add/list/rm — not `remove`), `openclaw config` (get/set/unset/file/validate), `openclaw onboard` all exist per `audit/_baseline.md` §3/§4C.7 (note: `cron remove` is `cron rm`).
- **Scheduling**: `openclaw cron add --name advisory-nightly --cron "30 4 * * *"` — use `--session isolated` so the 8-persona analysis doesn't pollute main chat context
- **Config**: `openclaw config set` for scalars; for the 14 data source configs edit JSON or workspace files
- **Model**: agent default (`agents.defaults.model` in `openclaw.json`). Override this skill's LLM routing via two env vars (distinct call tiers): **`COUNCIL_PERSONA_MODEL`** (per-persona calls, 8 parallel — cheap) and **`COUNCIL_SYNTHESIS_MODEL`** (single final synthesis — capable). See `_authoring/_deploy-common.md` "Per-skill env-var overrides" for the pattern + examples. L44 previously hardcoded `ANTHROPIC_API_KEY` — now sourced from `auth-profiles.json` based on provider routing.
- **Cost warning**: 8 parallel AI personas = expensive per run. Heartbeat model override is broken (GitHub #9556) so you cannot config-route to cheap model for this skill's analysis. Schedule sparingly or route through local Ollama as primary.
- **ClawHub alternative**: `council-of-the-wise` (multi-perspective spawn, §B) — different scope but adjacent

## Purpose

Install a nightly business analysis system with 8 parallel independent AI expert personas. Aggregates data from up to 14 sources, runs expert analysis in parallel (so they cannot influence each other), synthesizes findings into ranked recommendations delivered to Telegram.

**When to use:** After multiple data sources are deployed (CRM, social tracking, integrations). The more data sources connected, the more valuable the analysis.

**What this skill does:**
1. Creates the advisory council directory structure and ensures dependencies are available
2. Initializes the SQLite database for analysis artifacts, recommendations, and feedback
3. Configures data source connections and AI provider keys
4. Populates initial data via sync jobs
5. Runs a dry-run analysis to validate the full pipeline
6. Schedules nightly analysis and periodic sync cron jobs
7. Configures Telegram delivery and deeper-dive interaction

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Model Routing

This skill calls through to an LLM 8 times per run (one per persona) plus 1 synthesis call. It does NOT hardcode a provider. It defers to `agents.defaults.model` in `~/.openclaw/openclaw.json` (Codex-first per `audit/_baseline.md` §BB), which means:

- The active model is whatever `openclaw models set` / the agent's default resolves to
- Credentials live in `~/.openclaw/agents/main/agent/auth-profiles.json` (never in openclaw.json, never in .env)
- If you want to route this specific skill to a cheaper model, pass `--model <provider>/<model>` when you invoke `openclaw agent` (see Steps below) — don't try to use `agents.defaults.heartbeat.model` (known broken, GitHub #9556)

## Variables

Resolve these before executing. Sources are listed for each.

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | `agents.defaults.workspace` | `~/.openclaw/workspace` |
| `${TIMEZONE}` | Client profile | `America/Los_Angeles` |
| `${HUBSPOT_TOKEN}` | Client `.env` or `auth-profiles.json` (optional) | `pat-na1-...` |
| `${BEEHIIV_API_KEY}` | Client `.env` or `auth-profiles.json` (optional) | `bh_...` |
| `${ASANA_TOKEN}` | Client `.env` or `auth-profiles.json` (optional) | `1/12345...` |
| `${META_ANALYSIS_TOPIC_ID}` | Messaging setup | `1827` |
| `${ANALYSIS_SCHEDULE}` | Client preference (default: `30 4 * * *`) | `30 4 * * *` |

No `${ANTHROPIC_API_KEY}`, no `${BEARER}`, no `${SESSION_ID}` — LLM provider is resolved through `auth-profiles.json` at runtime, and there's no operator session to manage at this layer.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**

Check if the advisory council is already deployed:

**Remote:**
```
ls ${WORKSPACE}/tools/business-meta-analysis/ 2>/dev/null || echo 'NOT_DEPLOYED'
```

Check which AI provider is configured:

**Remote:**
```
openclaw config get models.primary 2>/dev/null || echo 'NO_MODEL_CONFIG'
```

Check how many data sources are available by looking for deployed systems:

**Remote:**
```
ls ${WORKSPACE}/data/*.db 2>/dev/null | head -20
```

**Decision points from pre-flight:**
- How many data sources are available? The council improves with more data, but works with as few as 3-4 sources.
- Which expert personas are relevant? Not every client needs all 8.
- What does the agent's default model stack resolve to? (`openclaw models status`). Advisory council inherits this — Codex-first per `audit/_baseline.md` §BB, though any provider with a valid auth profile works.
- Is this a redeployment? If the directory and database already exist, check if an update or fresh install is appropriate.

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Expert personas | All 8 (RevenueGuardian, GrowthStrategist, SkepticalOperator, TeamDynamicsArchitect, AutomationScout, CFO, ContentStrategist, MarketAnalyst) | Client does not need all domains. Solo creator might skip TeamDynamicsArchitect. Non-content business might skip ContentStrategist. |
| Data sources | 14 sources (YouTube, IG, X, CRM, email, Fathom, cron, Slack, Asana, HubSpot, Beehiiv, financials, etc.) | Only connected sources are used. Council gracefully handles missing sources. Deploy with what is available. |
| AI provider | Agent default (Codex-first per §BB) | Override per-skill via `COUNCIL_PERSONA_MODEL` / `COUNCIL_SYNTHESIS_MODEL` env vars (e.g., route personas to `openai-codex/gpt-5.4-mini` and synthesis to `openai-codex/gpt-5.4` for cost tuning) |
| Run schedule | Nightly 4:30am PST (`30 4 * * *`) | Client prefers different time, weekly instead of daily, or manual-only |
| Notification channel | Telegram Meta-Analysis topic | Client uses Discord, Slack, or email digest instead |
| Financial confidentiality | Directional only (no specific dollar amounts) | Client wants specifics delivered via DM |
| Sync schedules | Slack 3h, Asana 4h, HubSpot 4h | Cost, API limits, or data freshness needs differ |
| Deeper dives | "Tell me more about #N" in Telegram | Client prefers CLI-only or a different channel |

## Prerequisites

- OpenClaw installed and running (`openclaw/install.md`)
- `deploy-messaging-setup.md` completed (Telegram Meta-Analysis topic exists)
- `deploy-security-safety.md` completed
- At least some data sources deployed (more is better):
  - `deploy-personal-crm.md` -- CRM contacts, email activity
  - `deploy-social-tracking.md` -- YouTube, Instagram, X/Twitter analytics
  - `deploy-newsletter-crm.md` -- Beehiiv + HubSpot data (optional)
  - `deploy-asana.md` -- Task status (optional)
- A valid auth profile for the agent's default model (run `openclaw models auth list` to verify; re-auth with `openclaw models auth login --provider <x>` if needed)
- Node.js dependencies: The advisory council scripts require npm packages. The operator must ensure a `package.json` exists in the `business-meta-analysis` directory (or that required packages are otherwise available). If the project does not yet have a `package.json`, create one with the necessary dependencies before running `npm install`.

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
- All 8 run in parallel (independent -- cannot influence each other)
- Synthesizer merges findings, eliminates duplicates, ranks by priority
- Numbered digest delivered to Telegram
- "Tell me more about #3" triggers deeper dives
- Feedback loop: approve/reject recommendations, system learns preferences

### Database (`${WORKSPACE}/data/business-meta-analysis.db`)

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

Nightly Business Meta Analysis -- Daily at `${ANALYSIS_SCHEDULE}` in `${TIMEZONE}`

### Telegram Integration

- Meta-Analysis topic (ID: `${META_ANALYSIS_TOPIC_ID}`) for numbered digests and deeper dives

## Steps

### Phase 1: Directory Structure and Dependencies

#### 1.1 Create Advisory Council Directory Tree `[AUTO]`

Create the directory structure for the advisory council scripts, sync jobs, and persona definitions.

**Remote:**
```
mkdir -p ${WORKSPACE}/tools/business-meta-analysis/sync ${WORKSPACE}/tools/business-meta-analysis/scripts ${WORKSPACE}/tools/business-meta-analysis/personas
```

Expected: Three directories created under `${WORKSPACE}/tools/business-meta-analysis/`.

If already exists: Safe to re-run. `mkdir -p` is idempotent.

#### 1.2 Ensure Node.js Dependencies Are Available `[GUIDED]`

The advisory council scripts require npm packages to run. Ensure a `package.json` exists in the project directory with the necessary dependencies, then install them.

**Note:** If the workspace already has a shared `package.json` at `${WORKSPACE}/package.json` that covers the required packages, this step may be unnecessary. Check what already exists before creating a new one.

**Remote:**
```
ls ${WORKSPACE}/tools/business-meta-analysis/package.json 2>/dev/null || echo 'NO_PACKAGE_JSON'
```

If `NO_PACKAGE_JSON`: The operator must create an appropriate `package.json` for this project's dependencies (e.g., `better-sqlite3`, `node-fetch`, or whichever packages the analysis scripts require), then run:

**Remote:**
```
cd ${WORKSPACE}/tools/business-meta-analysis && npm install
```

Expected: `node_modules/` directory created with required packages. No errors.

If this fails: Check that Node.js and npm are on PATH. Check that the `package.json` has valid JSON. If behind a proxy, configure npm proxy settings.

If already exists: Check `node_modules/` exists and is not empty. If packages are outdated, `npm install` is safe to re-run.

### Phase 2: Configure Data Sources and AI Provider

#### 2.1 Set AI Provider API Key `[HUMAN_INPUT]`

**Model routing for this skill is handled through OpenClaw's existing model stack — there is no per-skill API key to configure.** The 8 persona calls run through `openclaw agent --agent main -m "<persona prompt>"`, which resolves to whichever model `agents.defaults.model` points at (Codex by default per `audit/_baseline.md` §BB). Credentials are already in `~/.openclaw/agents/main/agent/auth-profiles.json` from install time.

If you want this skill to use a DIFFERENT model than the agent default (e.g., route to a cheap model for the 8 parallel calls and the expensive synthesis call to a capable one), pass `--model <provider>/<model>` explicitly in the persona invocations inside `council.js`:

```javascript
// In scripts/council.js, persona-loop:
const PERSONA_MODEL = process.env.COUNCIL_PERSONA_MODEL || '';  // '' = use agent default
const flag = PERSONA_MODEL ? ['--model', PERSONA_MODEL] : [];
await openclawAgent(['--agent', 'main', ...flag, '-m', personaPrompt]);
```

Then set the env in the cron add step:
```
openclaw cron add --name advisory-nightly --cron '30 4 * * *' --session isolated \
  --command '${WORKSPACE}/tools/business-meta-analysis/run-council.sh' \
  --env COUNCIL_PERSONA_MODEL=openai-codex/gpt-5.4-mini \
  --env COUNCIL_SYNTHESIS_MODEL=openai-codex/gpt-5.4
```

Skip the env flags if the agent default is fine.

#### 2.2 Configure Optional Data Source Tokens `[HUMAN_INPUT]`

For each available integration, set the API token. Skip any that the client has not deployed.

**HubSpot (if `deploy-newsletter-crm.md` is deployed):**

**Remote:**
```
openclaw config set advisory-council.hubspot-token '${HUBSPOT_TOKEN}'
```

**Beehiiv (if `deploy-newsletter-crm.md` is deployed):**

**Remote:**
```
openclaw config set advisory-council.beehiiv-api-key '${BEEHIIV_API_KEY}'
```

**Asana (if `deploy-asana.md` is deployed):**

**Remote:**
```
openclaw config set advisory-council.asana-token '${ASANA_TOKEN}'
```

Expected: Each config value stored successfully.

If this fails: Verify the token is valid and not expired. HubSpot tokens expire and may need rotation. Asana personal access tokens do not expire but can be revoked.

If already exists: Config set is idempotent -- it overwrites the previous value. Safe to re-run with updated tokens.

### Phase 3: Initialize Data via Sync Jobs

#### 3.1 Run Initial Sync Jobs `[GUIDED]`

Run each sync manually to establish baseline data before the first analysis. Only run syncs for data sources that have been configured.

**Slack sync (if Slack is configured):**

**Remote:**
```
cd ${WORKSPACE}/tools/business-meta-analysis && node sync/slack-sync.cjs
```

**Asana sync (if Asana is configured):**

**Remote:**
```
cd ${WORKSPACE}/tools/business-meta-analysis && node sync/asana-sync.cjs
```

**HubSpot sync (if HubSpot is configured):**

**Remote:**
```
cd ${WORKSPACE}/tools/business-meta-analysis && node sync/hubspot-sync.cjs
```

Expected: Each sync completes without errors and reports the number of records ingested. First run may take longer as it pulls historical data.

If this fails: Check that the corresponding API token is valid and the service is reachable. Common causes: expired tokens (HubSpot, Asana), rate limits (Slack). Check sync script output for specific error messages.

### Phase 4: Validate the Pipeline

#### 4.1 Run a Dry-Run Analysis `[GUIDED]`

Test the full pipeline without delivering to Telegram. This validates that all 8 personas produce analysis and the synthesizer ranks recommendations correctly.

**Remote:**
```
cd ${WORKSPACE}/tools/business-meta-analysis && node scripts/nightly-business-meta-analysis.js --dry-run
```

Expected: Output shows analysis from all active personas (may be fewer than 8 if some data sources are not connected), plus a synthesized ranked recommendation list. No Telegram message sent.

If this fails:
- **API errors:** Check the agent's resolved model stack (`openclaw models status`) and confirm the target provider has a valid auth profile (`openclaw models auth list`). All 8 personas + synthesis call route through `agents.defaults.model` unless overridden via `COUNCIL_PERSONA_MODEL` / `COUNCIL_SYNTHESIS_MODEL` env.
- **Empty analysis:** Not enough data sources connected. The council works with 3-4 sources minimum. Deploy more data systems first if needed.
- **Script errors:** Check that dependencies are installed (`ls ${WORKSPACE}/tools/business-meta-analysis/node_modules/`).

#### 4.2 Test Deeper Dive Interaction `[GUIDED]`

Verify that deeper-dive requests work by requesting expansion on a recommendation from the dry run.

**Remote:**
```
cd ${WORKSPACE}/tools/business-meta-analysis && node scripts/council-deeper-dive.js --council advisory --number 1
```

Expected: Expanded analysis for recommendation #1 from the most recent run. The output should be more detailed than the digest summary.

If this fails: The referenced recommendation number must exist from the most recent dry run. Numbers reset each run. If the dry run produced fewer recommendations, use a valid number.

**Note (bug fix):** The council flag must be `--council advisory`, not `--council platform`. The advisory council is the correct council name for this system.

### Phase 5: Schedule Cron Jobs

#### 5.1 Schedule Nightly Analysis `[GUIDED]`

Register the nightly analysis cron job at the client's preferred time and timezone.

**Remote:**
```
openclaw cron add --name 'Nightly Business Meta Analysis' --schedule '${ANALYSIS_SCHEDULE}' --tz '${TIMEZONE}' --command 'cd ${WORKSPACE}/tools/business-meta-analysis && node scripts/nightly-business-meta-analysis.js'
```

Expected: Cron job registered. Verify with `openclaw cron list | grep -i "meta analysis"`.

If this fails: Check that `openclaw cron` is functional (`openclaw cron list`). If the cron system is not initialized, run `openclaw onboard --install-daemon`.

If already exists: Check for duplicate cron entries before adding. `openclaw cron list | grep -i "meta analysis"` -- if found, update the existing entry or remove the old one first.

#### 5.2 Schedule Sync Jobs `[AUTO]`

Register periodic sync cron jobs for each configured data source.

**Slack sync (every 3 hours):**

**Remote:**
```
openclaw cron add --name 'Slack Sync' --schedule '0 */3 * * *' --tz '${TIMEZONE}' --command 'cd ${WORKSPACE}/tools/business-meta-analysis && node sync/slack-sync.cjs'
```

**Asana sync (every 4 hours):**

**Remote:**
```
openclaw cron add --name 'Asana Sync' --schedule '0 */4 * * *' --tz '${TIMEZONE}' --command 'cd ${WORKSPACE}/tools/business-meta-analysis && node sync/asana-sync.cjs'
```

**HubSpot sync (every 4 hours):**

**Remote:**
```
openclaw cron add --name 'HubSpot Sync' --schedule '0 */4 * * *' --tz '${TIMEZONE}' --command 'cd ${WORKSPACE}/tools/business-meta-analysis && node sync/hubspot-sync.cjs'
```

Expected: Each cron job registered. Verify with `openclaw cron list`.

If already exists: Check for duplicate cron entries before adding. Only add sync jobs for data sources that are actually configured.

### Phase 6: Configure Telegram Delivery

#### 6.1 Set Telegram Topic for Digest Delivery `[AUTO]`

Configure the Telegram topic where numbered digests and deeper-dive responses will be delivered.

**Remote:**
```
openclaw config set advisory-council.telegram-topic ${META_ANALYSIS_TOPIC_ID}
```

Expected: Config value stored. Digests will be delivered to the Meta-Analysis topic.

If this fails: Verify the topic ID is correct. Check `deploy-messaging-setup.md` output or client profile for the Meta-Analysis topic ID.

If already exists: Config set is idempotent. Safe to re-run with the same or updated topic ID.

## Verification

Run these checks to confirm the advisory council is operational:

**Check sync status:**

**Remote:**
```
cd ${WORKSPACE}/tools/business-meta-analysis && node sync/hubspot-sync.cjs --status
```

Expected: Shows last sync time and record counts for each source.

**Run full dry-run with JSON output:**

**Remote:**
```
cd ${WORKSPACE}/tools/business-meta-analysis && node scripts/nightly-business-meta-analysis.js --dry-run --json
```

Expected: JSON output with persona analysis outputs and a synthesized ranked recommendation list.

**Verify cron registration:**

**Remote:**
```
openclaw cron list | grep -iE "meta analysis|slack sync|asana sync|hubspot sync"
```

Expected: All scheduled jobs appear with correct schedules.

**Verify Telegram delivery (optional, requires live run):**

After the first nightly run (or manually trigger with the analysis script without `--dry-run`), check the Telegram Meta-Analysis topic for a numbered digest.

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Empty analysis | Not enough data sources connected | Deploy more data systems first. Even 3-4 sources is enough to start. |
| API errors during analysis | Auth profile missing or invalid for the routed model | `openclaw models status` to see what resolves; `openclaw models auth list` to see profiles; re-auth with `openclaw models auth login --provider <x>` if the profile is stale |
| `npm install` fails or no `package.json` | Dependencies not set up | Create a `package.json` with required dependencies in the `business-meta-analysis` directory, then re-run `npm install`. |
| Sync failures | Individual sync job errors | Check sync job logs in cron-log. Most common: expired tokens (HubSpot, Asana) or rate limits (Slack). |
| Confidentiality violation | Financial data too specific in digest | The CFO persona uses directional language only. If specific amounts appear, check the CFO persona prompt in `${WORKSPACE}/tools/business-meta-analysis/personas/`. |
| Deeper dive returns nothing | Referenced recommendation number does not exist | Use the exact number from the most recent digest. Numbers reset each run. |
| Deeper dive with `--council platform` | Wrong council name | Use `--council advisory`, not `--council platform`. |
| Personas influencing each other | Architecture bug | All 8 must run in parallel with isolated contexts. Check that the runner uses `Promise.all()` and not sequential execution. |
| Duplicate cron entries | Skill run multiple times | Check `openclaw cron list` for duplicates. Remove extras with `openclaw cron rm`. |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Create directories | Execute silently | Execute silently | Confirm before creating |
| 1.2 Install dependencies | Execute silently | Confirm before install | Confirm each command |
| 2.1 Set AI provider key | Prompt for key | Prompt for key | Prompt for key |
| 2.2 Configure data source tokens | Prompt for each token | Prompt for each token | Prompt for each token |
| 3.1 Run initial syncs | Execute silently | Confirm before running | Confirm each sync |
| 4.1 Dry-run analysis | Execute, review output | Confirm, review output together | Confirm, step through output |
| 4.2 Test deeper dive | Execute silently | Confirm before running | Confirm before running |
| 5.1 Schedule nightly analysis | Execute silently | Confirm schedule and timezone | Confirm each detail |
| 5.2 Schedule sync jobs | Execute silently | Execute silently | Confirm each sync job |
| 6.1 Set Telegram topic | Execute silently | Execute silently | Confirm topic ID |

## Dependencies

- **Depends on:** `openclaw/install.md`, `deploy-messaging-setup.md`, `deploy-security-safety.md`
- **Enhanced by:** `deploy-personal-crm.md`, `deploy-social-tracking.md`, `deploy-newsletter-crm.md`, `deploy-asana.md`, `deploy-fathom-pipeline.md`
- **Required by:** Nothing directly, but the advisory council's recommendations can feed into daily briefings and other decision-support systems.
