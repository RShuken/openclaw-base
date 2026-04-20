---
name: "Deploy Video Idea Pipeline"
category: "integration"
subcategory: "content-gen"
third_party_service: "Slack + Asana + X/Twitter"
auth_type: "api-key"
openclaw_version_required: "2026.4.15+"
version_last_verified: "2026-04-20"
last_checked: "2026-04-20"
maintenance_status: "active"
model_env_var: "VIDEO_PIPELINE_MODEL"
clawhub_alternative: "none"
cost_tier: "api-usage"
privacy_tier: "cloud"
requires:
  - "Asana Personal Access Token"
  - "Slack workspace + channel (openclaw channels)"
  - "X/Twitter API access"
  - "SQLite with vector embeddings"
  - "Knowledge base (deploy-knowledge-base.md)"
docs_urls:
  - "https://api.slack.com/messaging/retrieving"
  - "https://developers.asana.com/reference/rest-api-reference"
---

# Deploy Video Idea Pipeline

## Compatibility

- **OpenClaw Version**: 2026.4.15+ (compat header rewritten 2026-04-20 — live-verified on 4C)
- **Status**: WORKING — half the previous "blocked commands" list was mislabels, the other half never existed:
  - `openclaw config get/set` — **VALID** on 2026.4.15 per `audit/_baseline.md` §4C.7 (was falsely flagged blocked)
  - `openclaw skill install` — **TYPO for `openclaw skills install`** which IS valid per §4C.4
  - `openclaw slack status` — never existed; Slack goes through `openclaw channels` (§4C.7) or raw Slack API
  - `openclaw asana list-tasks` / `openclaw asana test` — never existed; Asana goes through direct API calls (see `deploy-asana.md`)
- **Scheduling**: `openclaw cron add --name video-pipeline --cron "0 9 * * 1"` for weekly Monday 9am
- **Model**: agent default (`agents.defaults.model` in `openclaw.json`). Override this skill's LLM routing via env var **`VIDEO_PIPELINE_MODEL`** (see `_authoring/_deploy-common.md` "Per-skill env-var overrides" for the pattern + examples). Applies to pipeline synthesis calls (thread read, research digest, Asana card draft).
- **Integrations**: Slack via `openclaw channels` (or raw API); Asana via direct API (see `deploy-asana.md` for the pattern)

## Purpose

Install an automated video idea research pipeline triggered by Slack mentions. When a user says "@assistant potential video idea" in a Slack thread, the agent reads the full thread, runs X/Twitter research, queries the knowledge base, and creates a structured Asana card with research findings. Includes a semantic dedup system backed by a SQLite pitch database with vector embeddings to prevent recycled ideas.

**When to use:** When the client creates video content and wants automated idea research and tracking.

**What this skill does:**
1. Installs the video idea pipeline skill (Slack trigger + research + card creation)
2. Installs the X/Twitter research skill (bun-based, requires X API access)
3. Configures Asana as the card destination for video ideas
4. Initializes the video pitches SQLite database with vector embeddings for semantic dedup
5. Configures the Slack trigger phrase and verifies end-to-end flow

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

Resolve these before executing. Sources are listed for each.

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${ASANA_PAT}` | Client or client profile: Asana personal access token | `1/12345:abcdef...` |
| `${ASANA_VIDEO_PROJECT_ID}` | Client profile or Asana project list: the Video Pipeline project ID | `1212455754265217` |
| `${X_BEARER_TOKEN}` | Client profile or X Developer Portal: API bearer token | `AAAA...` |
| `${GEMINI_API_KEY}` | Client profile: Gemini API key (used for embeddings) | `AIza...` |
| `${SLACK_TRIGGER_PHRASE}` | Client preference (default: `@assistant potential video idea`) | `@assistant potential video idea` |
| `${DEDUP_THRESHOLD}` | Client preference (default: `40`) | `40` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**

Check if the knowledge base skill is deployed (used for research queries):

**Remote:**
```
ls ${WORKSPACE}/skills/knowledge-base/ 2>/dev/null && echo 'KB_FOUND' || echo 'KB_NOT_FOUND'
```

Check if x-research-v2 is installed:

**Remote:**
```
ls ${WORKSPACE}/skills/x-research-v2/ 2>/dev/null && echo 'XRESEARCH_FOUND' || echo 'XRESEARCH_NOT_FOUND'
```

Check if Asana PAT is configured:

**Remote:**
```
echo $ASANA_PAT | head -c 8
```

Check if the video pipeline is already deployed:

**Remote:**
```
ls ${WORKSPACE}/skills/video-idea-pipeline/ 2>/dev/null && echo 'PIPELINE_FOUND' || echo 'PIPELINE_NOT_FOUND'
```

Check if the video pitches database already exists:

**Remote:**
```
ls ${WORKSPACE}/data/video-pitches.db 2>/dev/null && echo 'DB_FOUND' || echo 'DB_NOT_FOUND'
```

Check Slack connection status:

**Remote:**
```
openclaw channels list | grep -i slack
```

**Decision points from pre-flight:**
- Is the knowledge base deployed? If not, deploy `deploy-knowledge-base.md` first (or skip KB research if client prefers).
- Is x-research-v2 installed? If not, it will be installed in Phase 2 (or skip if client has no X API access).
- Is Asana connected? If not, deploy `deploy-asana.md` first (or use an alternative card destination).
- Is Slack connected? If not, deploy `deploy-messaging-setup.md` first (or use an alternative trigger).
- Does the pitches database already exist? If so, skip initialization.

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Alternative | When to Adapt |
|-----------|---------|-------------|---------------|
| Trigger platform | Slack (`@assistant potential video idea`) | Telegram command, Discord, CLI | Client doesn't use Slack |
| Research: X/Twitter | x-research-v2 via bun + X API | Skip X research, use web search instead | Client doesn't have X API access |
| Research: Knowledge base | KB semantic search | Skip if KB not deployed | Knowledge base not installed |
| Card destination | Asana (Video Pipeline project) | Linear, Notion, Trello, local markdown | Client uses a different project management tool |
| Dedup threshold | 40% semantic similarity = auto-skip | Adjust threshold | Client wants stricter/looser dedup |
| Pitch database | SQLite with embeddings at `${WORKSPACE}/data/video-pitches.db` | Same (no good alternative) | Rarely needs changing |
| Feedback learning | Learns from accept/reject | Manual curation only | Client preference |

## Prerequisites

- OpenClaw installed and running (`openclaw/install.md`)
- `deploy-knowledge-base.md` completed (research queries)
- `deploy-messaging-setup.md` completed (Slack connection)
- `deploy-asana.md` completed (Asana task creation)
- `${GEMINI_API_KEY}` configured in `~/.openclaw/.env` (used for pitch embeddings via gemini-embedding-001)

## What Gets Installed

### Video Idea Pipeline Skill (`${WORKSPACE}/skills/video-idea-pipeline/`)

The main pipeline orchestrator. Triggered by Slack mention, runs research, creates Asana cards.

- **Trigger:** User says the trigger phrase in Slack
- **Research:** Full thread read -> X/Twitter research -> KB query
- **Output:** Structured Asana card with idea, research, sources, angles
- **Completion:** Message posted back in Slack thread with Asana link

### Video Pitches Database (`${WORKSPACE}/data/video-pitches.db`)

SQLite database with vector embeddings for semantic search and dedup.

| Table | Purpose |
|-------|---------|
| pitches | Core pitch data: title, description, source, status, embeddings |
| pitch_research | Research findings per pitch: sources, angles, quotes |
| pitch_feedback | Accept/reject feedback for learning |

| Column (pitches) | Purpose |
|-------------------|---------|
| id | Auto-incrementing primary key |
| slug | URL-friendly identifier (auto-generated) |
| title | Pitch title |
| description | Full pitch description |
| source | Where the idea came from (slack thread URL) |
| status | pitched, accepted, rejected, produced, duplicate |
| embedding | 768-dim vector for semantic similarity |
| similarity_score | Highest similarity to existing pitches (for dedup) |
| created_at | Timestamp |

Semantic dedup: 40%+ similarity -> auto-skip (prevents recycled ideas).

Learning: Accept/reject feedback improves future research quality and angle suggestions.

### Pitch Management Scripts (`${WORKSPACE}/skills/video-pitches/`)

| Script | Purpose |
|--------|---------|
| add.js | Add a pitch with embeddings, auto-ID, auto-slug |
| search.js | Semantic + keyword search (70% cosine, 30% keyword) |
| list.js | List recent pitches with filters |
| update.js | Update status |

### Pipeline Flow

1. User mentions the trigger phrase in Slack
2. Agent reads the full Slack thread for context
3. X/Twitter research runs on the topic (trending angles, audience sentiment)
4. Knowledge base queried for related content and past coverage
5. Semantic dedup check against existing pitches (threshold configurable)
6. If not a duplicate: create structured Asana card in Video Pipeline project
7. Post completion message back in Slack thread with Asana link
8. If duplicate: notify in Slack with link to the existing similar pitch

## Steps

### Phase 1: Install Pipeline Skill

#### 1.1 Install Video Idea Pipeline Skill `[GUIDED]`

Install the main pipeline skill that orchestrates the Slack trigger, research, and Asana card creation.

**Remote:**
```
openclaw skills install video-idea-pipeline --path ${WORKSPACE}/skills/video-idea-pipeline/
```

Expected: Skill directory created at `${WORKSPACE}/skills/video-idea-pipeline/` with pipeline configuration files.

If this fails: Check that OpenClaw is running (`openclaw health`). Check that `${WORKSPACE}/skills/` directory exists.

If already exists: Check the existing installation. If it appears correct, skip. If outdated or broken, back up the existing directory and reinstall.

### Phase 2: Install X/Twitter Research

#### 2.1 Ensure Bun Runtime is Available `[AUTO]`

The X/Twitter research skill requires bun as its JavaScript runtime.

**Remote:**
```
which bun || curl -fsSL https://bun.sh/install | bash
```

Expected: `bun` command available on PATH.

If this fails: On macOS, try `brew install oven-sh/bun/bun`. On Linux, try the install script again. On Windows, use `irm bun.sh/install.ps1 | iex`.

If already exists: Skip (the `which bun` check handles this automatically).

#### 2.2 Install X-Research-V2 Skill `[GUIDED]`

Install the X/Twitter research skill used by the pipeline for trending angles and audience sentiment.

**Remote:**
```
openclaw skills install x-research-v2 --path ${WORKSPACE}/skills/x-research-v2/
```

Expected: Skill directory created at `${WORKSPACE}/skills/x-research-v2/`.

If this fails: Check that `bun` is installed (Step 2.1). Check OpenClaw status.

If already exists: Verify the skill is functional by checking if x-research-v2 files exist. If so, skip.

#### 2.3 Verify X/Twitter Credentials `[HUMAN_INPUT]`

Confirm the X/Twitter bearer token is configured. Without it, X research will silently fail.

**Remote:**
```
grep X_BEARER_TOKEN ~/.openclaw/.env
```

Expected: Line showing `X_BEARER_TOKEN=<token>` (token should be non-empty).

If this fails: The client needs to provide their X API bearer token from the X Developer Portal. Set it:

**Remote:**
```
echo 'X_BEARER_TOKEN=${X_BEARER_TOKEN}' >> ~/.openclaw/.env
```

> **Note:** Substitute the actual token value before sending. If the client has no X API access, skip X research and note the adaptation -- the pipeline will still work using KB research alone.

Optional — to route the pipeline's LLM synthesis through a specific model, add `VIDEO_PIPELINE_MODEL` to `~/.openclaw/.env`:

```
echo 'VIDEO_PIPELINE_MODEL=openai-codex/gpt-5.4-nano' >> ~/.openclaw/.env
```

In the pipeline node scripts (`add.js`, `search.js`, and the Slack-event handler that drafts Asana cards):

```javascript
const MODEL = process.env.VIDEO_PIPELINE_MODEL || '';  // empty → agent default
const modelFlag = MODEL ? ['--model', MODEL] : [];
spawn('openclaw', ['agent', '--agent', 'main', ...modelFlag, '-m', prompt]);
```

### Phase 3: Configure Asana Integration

#### 3.1 Set Asana Personal Access Token `[HUMAN_INPUT]`

Configure the Asana PAT so the pipeline can create task cards.

**Remote:**
```
openclaw config set asana.pat '${ASANA_PAT}'
```

Expected: Config updated confirmation.

If this fails: Check that OpenClaw is running. Verify the PAT is valid.

If already configured: Check existing PAT with `openclaw config get asana.pat`. If it returns a value, skip.

#### 3.2 Set Video Pipeline Project ID `[GUIDED]`

Point the pipeline at the correct Asana project for video idea cards.

**Remote:**
```
openclaw config set asana.video-pipeline-project '${ASANA_VIDEO_PROJECT_ID}'
```

Expected: Config updated confirmation.

If this fails: Verify the project ID is correct by checking Asana. The project ID is the numeric ID from the Asana project URL.

#### 3.3 Verify Asana Connection `[AUTO]`

Test that the Asana integration is working and can reach the video pipeline project.

**Remote:**
```
openclaw asana test --project ${ASANA_VIDEO_PROJECT_ID}
```

Expected: Successful connection and project access confirmed.

If this fails: Check that `${ASANA_PAT}` is valid (tokens expire). Verify the project ID exists and the PAT has access to that project. Check Asana permissions.

### Phase 4: Initialize Pitch Database

#### 4.1 Initialize Video Pitches Database `[AUTO]`

Create the SQLite database with vector embeddings support for semantic dedup. This creates the tables (pitches, pitch_research, pitch_feedback) and sets up the embedding index.

**Remote:**
```
cd ${WORKSPACE}/skills/video-pitches && node add.js --init
```

Expected: Database created at `${WORKSPACE}/data/video-pitches.db` with all three tables.

If this fails: Check that `${WORKSPACE}/data/` directory exists (create with `mkdir -p ${WORKSPACE}/data`). Check that `${GEMINI_API_KEY}` is set in `~/.openclaw/.env` (needed for the embedding model). Verify Node.js is available.

If already exists: The `--init` flag should be idempotent (`CREATE TABLE IF NOT EXISTS`). Safe to re-run.

### Phase 5: Configure Slack Trigger

#### 5.1 Set Up Slack Trigger Phrase `[GUIDED]`

Configure the Slack listener to detect the video idea trigger phrase and activate the pipeline.

**Remote:**
```
openclaw config set slack.triggers.video-idea '${SLACK_TRIGGER_PHRASE}'
```

Expected: Config updated confirmation.

If this fails: Check that Slack socket mode is configured (`deploy-messaging-setup.md` must be completed first).

If already configured: Check existing trigger with `openclaw config get slack.triggers.video-idea`. If correct, skip.

#### 5.2 Verify Slack Connection `[AUTO]`

Confirm Slack socket mode is connected and listening for triggers.

**Remote:**
```
openclaw channels list | grep -i slack
```

Expected: Socket mode connected, bot active.

If this fails: Re-run `deploy-messaging-setup.md` Slack setup steps. Check that the Slack bot token is valid and socket mode is enabled in the Slack app settings.

### Phase 6: End-to-End Test

#### 6.1 Test with a Manual Pitch `[GUIDED]`

Add a test pitch directly to the database to verify the pitch management scripts work.

**Remote:**
```
cd ${WORKSPACE}/skills/video-pitches && node add.js --title 'Test Pitch' --description 'Testing the video idea pipeline' --source 'manual test'
```

Expected: Pitch added with auto-generated ID, slug, and embedding vector. Output should show the new pitch details.

If this fails: Check that the database was initialized (Step 4.1). Verify `${GEMINI_API_KEY}` is set (needed for embedding generation).

#### 6.2 Verify Pitch in Database `[AUTO]`

Confirm the test pitch was stored correctly.

**Remote:**
```
cd ${WORKSPACE}/skills/video-pitches && node list.js --recent 5
```

Expected: List showing the "Test Pitch" entry with status "pitched".

If this fails: Check that the database file exists at `${WORKSPACE}/data/video-pitches.db`.

#### 6.3 Verify Asana Task Creation `[AUTO]`

Check that the pipeline can create cards in the Asana project.

**Remote:**
```
openclaw asana list-tasks --project ${ASANA_VIDEO_PROJECT_ID} --limit 5
```

Expected: Task list from the Video Pipeline project (may or may not include the test pitch depending on whether the pipeline's Asana integration ran).

If this fails: Re-check Asana PAT and project ID (Steps 3.1-3.3).

#### 6.4 Test Semantic Search `[AUTO]`

Verify the semantic search and dedup system works.

**Remote:**
```
cd ${WORKSPACE}/skills/video-pitches && node search.js 'test topic'
```

Expected: Search results returned (may include the test pitch if the topic is similar enough).

If this fails: Check that embeddings were generated for the test pitch. Verify `${GEMINI_API_KEY}` is valid.

## Verification

Run these checks to confirm the video idea pipeline is fully operational:

**Pitch database is functional:**

**Remote:**
```
cd ${WORKSPACE}/skills/video-pitches && node list.js --recent 5
```

Expected: Recent pitches listed (at least the test pitch from Phase 6).

**Semantic search works:**

**Remote:**
```
cd ${WORKSPACE}/skills/video-pitches && node search.js 'test topic'
```

Expected: Search results returned with similarity scores.

**Slack is connected and listening:**

**Remote:**
```
openclaw channels list | grep -i slack
```

Expected: Socket mode connected, triggers configured.

**Full end-to-end test (optional but recommended):**

Post the trigger phrase (e.g., "@assistant potential video idea: best practices for short-form content") in a Slack channel and verify:
- Agent reacts with eyes emoji
- Research runs (X/Twitter + KB)
- Asana card created in Video Pipeline project
- Completion message posted back in Slack thread with Asana link
- Pitch logged in `${WORKSPACE}/data/video-pitches.db`

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Slack trigger not firing | Socket mode disconnected or bot not mentioned | Check `openclaw channels list \| grep -i slack`. Verify the bot is mentioned with the exact trigger phrase. |
| X/Twitter research empty | `X_BEARER_TOKEN` invalid or expired | Verify token: `grep X_BEARER_TOKEN ~/.openclaw/.env`. Test manually: `cd ${WORKSPACE}/skills/x-research-v2 && bun run research.js "test topic"`. |
| KB results empty | Knowledge base not populated | Ingest content into the knowledge base first via `deploy-knowledge-base.md`. |
| Asana card not created | `ASANA_PAT` invalid or project ID wrong | Verify PAT: `openclaw asana test`. Verify project ID matches `${ASANA_VIDEO_PROJECT_ID}`. Check Asana permissions. |
| Semantic dedup false positive | Similarity threshold too low | The default is 40%. If legitimate ideas are being flagged as duplicates, increase the threshold in the pipeline config. |
| Embedding errors | `GEMINI_API_KEY` not set | Verify `GEMINI_API_KEY` in `~/.openclaw/.env`. The pitch database uses gemini-embedding-001 for vectors. |
| All pitches marked "duplicate" | Database has too many broad pitches | Review existing pitches: `cd ${WORKSPACE}/skills/video-pitches && node list.js --status pitched`. Consider archiving or rejecting old generic pitches. |
| `add.js --init` fails | Missing data directory | Create it: `mkdir -p ${WORKSPACE}/data`. Then re-run init. |
| Bun not found | Bun not installed or not on PATH | Re-run: `curl -fsSL https://bun.sh/install | bash`. Refresh shell: `source ~/.bashrc`. |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Install pipeline skill | Execute | Confirm before install | Confirm |
| 2.1 Install bun runtime | Execute | Execute | Confirm |
| 2.2 Install x-research-v2 | Execute | Confirm before install | Confirm |
| 2.3 X/Twitter credentials | Pause for token input | Pause for token input | Pause for token input |
| 3.1 Set Asana PAT | Pause for PAT input | Pause for PAT input | Pause for PAT input |
| 3.2 Set project ID | Execute | Confirm project ID | Confirm |
| 3.3 Verify Asana | Execute | Execute | Show result, confirm |
| 4.1 Initialize database | Execute | Execute | Confirm |
| 5.1 Configure Slack trigger | Execute | Confirm trigger phrase | Confirm |
| 5.2 Verify Slack | Execute | Execute | Show result, confirm |
| 6.1-6.4 End-to-end test | Execute all, report results | Confirm before testing | Confirm each test |

## Dependencies

- **Depends on:** `openclaw/install.md`, `deploy-knowledge-base.md`, `deploy-messaging-setup.md`, `deploy-asana.md`
- **Required by:** None (leaf skill)
- **Enhanced by:** Richer knowledge base content improves research quality. More pitch feedback data improves future suggestions.
- **Requires skill:** x-research-v2 (installed during Phase 2)
