# Deploy Knowledge Base (RAG)

## Compatibility

- **OpenClaw Version**: 2026.4.15+ (compat header rewritten 2026-04-19 — live-verified on 4C)
- **Status**: WORKING — `openclaw config get/set` exist on 2026.4.15 per `audit/_baseline.md` §4C.7
- **Overlap with bundled `memory`**: OpenClaw 2026.4.15 ships a `memory` subsystem (top-level subcommand, uses sqlite-vec, auth-profiles-backed embeddings). For simple workspace-wide semantic search, use `openclaw memory` directly. This KB skill's value-add is URL/YouTube/X/PDF INGESTION on top of semantic search — don't rebuild what `memory` already does.
- **Config**: use `openclaw config set` for scalar KB settings; edit JSON directly for arrays
- **Model**: agent default (`agents.defaults.model` in `openclaw.json`). Override this skill's LLM routing via env var **`KB_QUERY_MODEL`** (see `_authoring/_deploy-common.md` "Per-skill env-var overrides" for the pattern + examples). Applies to the RAG query synthesis step in `scripts/query.js`.
- **ClawHub alternatives**: `private-knowledge-base`, `rag-search`, `hk101-living-rag`, `sulada-knowledge-base` — see `audit/_baseline.md` §B

## Purpose

Install a personal knowledge base with RAG (Retrieval-Augmented Generation). Ingest URLs via Telegram -- articles, YouTube transcripts, X/Twitter threads, PDFs. Supports semantic search with time-aware and source-weighted ranking.

**When to use:** When the client wants to save and query reference material from the web.

**What this skill does:**
1. Creates the knowledge base directory structure and installs Node.js dependencies
2. Installs content extraction tooling (summarize CLI or equivalent)
3. Initializes the SQLite database with sources, chunks, and entities tables
4. Tests the full ingestion and query pipeline
5. Configures Telegram topic integration for URL drop-to-ingest
6. Optionally configures Slack cross-posting for team visibility

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

Resolve these before executing. Sources are listed for each.

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${KB_TOPIC_ID}` | Created during messaging setup (`deploy-messaging-setup.md`) | `1173` |
| `${GEMINI_API_KEY}` | Client profile: API keys, or `~/.openclaw/.env` | `AIza...` |
| `${X_BEARER_TOKEN}` | Client profile: X/Twitter API bearer token (optional) | `AAAA...` |
| `${SLACK_BOT_TOKEN}` | Client profile: Slack bot token (optional) | `xoxb-...` |
| `${SLACK_CHANNEL}` | Client preference for cross-post channel (optional) | `ai_trends` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**

Check if the knowledge base database already exists (indicates prior deployment):

**Remote:**
```
ls ${WORKSPACE}/skills/knowledge-base/data/knowledge.db 2>/dev/null || echo 'NOT_FOUND'
```

Expected: `NOT_FOUND` for a fresh install, or the file path if upgrading.

Check if the `summarize` CLI is available (used for URL content extraction and YouTube transcripts):

**Remote:**
```
which summarize 2>/dev/null || echo 'NOT_INSTALLED'
```

Check if `bun` is available (needed for X/Twitter thread ingestion via x-research-v2):

**Remote:**
```
which bun 2>/dev/null || echo 'NOT_INSTALLED'
```

Check if Node.js and npm are available:

**Remote:**
```
node --version && npm --version
```

**Decision points from pre-flight:**
- Is there an existing knowledge base? If so, this is an upgrade -- back up the database before proceeding.
- Is the `summarize` CLI installed? If not, install it in Phase 2.
- Is `bun` available? If not, X/Twitter thread ingestion will be limited.
- Does the client want Slack cross-posting? (Optional, requires `SLACK_BOT_TOKEN`.)
- What embedding provider does the client prefer? (Default: Google Gemini.)

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Ingestion trigger | Telegram KB topic | Client uses Discord, Slack, or CLI-only |
| Embedding provider | Google `gemini-embedding-001` | Client prefers OpenAI `text-embedding-3-small` or another provider |
| X/Twitter ingestion | x-research-v2 via `bun` + X API | Client lacks X API access or `bun`; disable or use Grok fallback |
| Paywalled content | Browser automation via Chrome session | Client doesn't want browser automation; skip or use Firecrawl/Apify |
| Slack cross-post | Disabled by default | Client wants team visibility of ingested articles |
| Database path | `${WORKSPACE}/skills/knowledge-base/data/knowledge.db` | Client wants databases in a different location |
| Content extraction tool | `summarize` CLI (see Step 2.1) | Non-macOS platform or client prefers alternative tooling |

## Prerequisites

- OpenClaw installed and running (`openclaw/install.md`)
- `deploy-messaging-setup.md` completed (Telegram KB topic)
- `deploy-security-safety.md` completed (content sanitizer for web ingestion)
- Node.js + npm available
- `GEMINI_API_KEY` (or alternative embedding provider key) configured in `~/.openclaw/.env`
- Optional: `bun` runtime for X/Twitter thread research
- Optional: `SLACK_BOT_TOKEN` for cross-posting
- Optional: `X_BEARER_TOKEN` for X/Twitter API access

## What Gets Installed

### SQLite Database (`${WORKSPACE}/skills/knowledge-base/data/knowledge.db`)

| Table | Purpose |
|-------|---------|
| sources | Source metadata (URL, title, type, tags, ingestion timestamp, word count) |
| chunks | Content chunks with 768-dim vector embeddings (gemini-embedding-001) |
| entities | Extracted entities (people, companies, concepts) linked to sources |

### Supported Input Types

| Type | How It Works |
|------|-------------|
| Articles (any URL) | Fetch, extract content, summarize, embed |
| YouTube videos | Pull transcript via summarize CLI, embed |
| X/Twitter posts | Follow full threads automatically (not just first tweet), linked articles ingested too |
| PDFs | Extract text, chunk, embed |
| Paywalled sites | Browser automation via Chrome session (browser-control skill) |

### Scripts

| Script | Purpose |
|--------|---------|
| ingest.js | Ingest a single URL with tags, title override, type override |
| ingest-and-crosspost.js | Ingest + optional Slack cross-post |
| query.js | Natural language query with semantic search |
| list.js | List sources with tag/type/recent filters |
| delete.js | Delete a source by ID |
| stats.js | Database statistics |
| bulk-ingest.js | Bulk ingest from file |

### Search Features

- Semantic search via vector embeddings
- Time-aware ranking (recent sources rank higher)
- Source-weighted ranking
- Entity extraction for structured queries

## Steps

### Phase 1: Directory Structure and Dependencies

#### 1.1 Create KB Skill Directory `[AUTO]`

Create the directory structure for the knowledge base skill, including subdirectories for data, scripts, and source code.

**Remote:**
```
mkdir -p ${WORKSPACE}/skills/knowledge-base/data ${WORKSPACE}/skills/knowledge-base/scripts ${WORKSPACE}/skills/knowledge-base/src
```

Expected: Directories created without errors.

If this fails: Check that `${WORKSPACE}` exists. If not, the workspace hasn't been initialized -- run `openclaw/install.md` first.

If already exists: `mkdir -p` is idempotent. Safe to re-run.

#### 1.2 Install Node.js Dependencies `[AUTO]`

Install the npm packages required by the knowledge base scripts (SQLite bindings, embedding client, HTTP fetch, etc.).

**Remote:**
```
cd ${WORKSPACE}/skills/knowledge-base && npm install
```

Expected: `node_modules/` populated, no errors.

If this fails: Check that `package.json` exists in `${WORKSPACE}/skills/knowledge-base/`. If this is a fresh deployment, you may need to write the `package.json` first (check the skill's source repo or template). Also verify Node.js and npm are on PATH.

If already exists: `npm install` is idempotent. It will update any changed dependencies.

### Phase 2: Content Extraction Tooling

#### 2.1 Install Content Extraction CLI `[GUIDED]`

Install the `summarize` CLI, which handles URL content extraction and YouTube transcript pulling. This is the primary tool for converting web content into text suitable for chunking and embedding.

**Note:** The `summarize` CLI (`steipete/tap/summarize`) is a macOS Homebrew tap. For other platforms, use alternatives or skip this step (reduces YouTube transcript capability).

**Remote (macOS with Homebrew):**
```
brew install steipete/tap/summarize
```

**Alternative (any platform with npm):**
If `summarize` is unavailable, you can use `readability-cli` or similar tools for URL content extraction:
```
npm install -g @nicholasgriffintn/readability-cli
```

**Alternative (YouTube-only transcripts):**
For YouTube transcript extraction without the full summarize CLI:
```
npm install -g youtube-transcript-api
```

**Cross-platform note:** If the client is on Linux or Windows, discuss which content extraction approach fits best. The core knowledge base works without `summarize` -- it just means YouTube transcripts and some content extraction features may be limited or require alternative tooling.

Expected: Content extraction tool available on PATH.

**Remote:**
```
which summarize 2>/dev/null && summarize --version || echo 'Using alternative extraction'
```

If this fails (macOS): Ensure Homebrew is installed (`which brew`). If Homebrew is missing, install it first or use the npm alternative.

If this fails (Linux/Windows): This is expected. Use the npm alternatives above or skip and note the limitation.

If already exists: `brew install` will report "already installed." Safe to re-run.

### Phase 3: Database Initialization

#### 3.1 Verify Database Path `[AUTO]`

Confirm the data directory is ready. The database itself is auto-created on first ingest, but the directory must exist.

**Remote:**
```
ls -la ${WORKSPACE}/skills/knowledge-base/data/
```

Expected: Directory listing (may be empty on fresh install, or contain `knowledge.db` on upgrade).

If this fails: Directory wasn't created in Phase 1. Re-run step 1.1.

#### 3.2 Back Up Existing Database (Upgrade Only) `[GUIDED]`

If an existing `knowledge.db` was found during pre-flight, back it up before proceeding.

**Remote:**
```
cp ${WORKSPACE}/skills/knowledge-base/data/knowledge.db ${WORKSPACE}/skills/knowledge-base/data/knowledge.db.bak.$(date +%Y%m%d%H%M%S)
```

Expected: Backup file created with timestamp suffix.

If this fails: Check disk space and file permissions.

If already exists: Each backup has a unique timestamp. Safe to run multiple times.

### Phase 4: Test the Pipeline

#### 4.1 Test Ingestion `[GUIDED]`

Ingest a test article to confirm the full pipeline works: fetch, extract content, chunk, generate embeddings, and store in SQLite.

**Remote:**
```
cd ${WORKSPACE}/skills/knowledge-base && node scripts/ingest.js 'https://example.com/article' --tags test
```

Expected: Output showing successful ingestion -- URL fetched, content extracted, chunks created, embeddings stored. The source should appear in the database.

If this fails:
- **"summarize: command not found"**: Content extraction CLI not installed (Phase 2).
- **"GEMINI_API_KEY" error**: Embedding API key not configured. Check `~/.openclaw/.env`.
- **Network error on fetch**: Check internet connectivity on the client machine.
- **SQLite error**: Check that the `data/` directory is writable.
- **Lock file error**: A stale lock file from a crashed process. Remove `${WORKSPACE}/skills/knowledge-base/data/.ingest.lock` if no ingestion is actually running.

#### 4.2 Test Query `[GUIDED]`

Run a semantic search query against the ingested test content to verify the retrieval pipeline.

**Remote:**
```
cd ${WORKSPACE}/skills/knowledge-base && node scripts/query.js 'What was that article about?'
```

To route the RAG synthesis through a specific model, prefix with the env var:

```
KB_QUERY_MODEL=openai-codex/gpt-5.4-nano node scripts/query.js 'What was that article about?'
```

In `scripts/query.js`:

```javascript
const MODEL = process.env.KB_QUERY_MODEL || '';  // empty → agent default
const modelFlag = MODEL ? ['--model', MODEL] : [];
spawn('openclaw', ['agent', '--agent', 'main', ...modelFlag, '-m', synthesisPrompt]);
```

Expected: Ranked results with relevance scores referencing the test article.

If this fails:
- **No results**: Ingestion may not have completed successfully. Check `node scripts/stats.js` for source/chunk counts.
- **Embedding error**: Verify `GEMINI_API_KEY` is valid and has quota remaining.

### Phase 5: Telegram Integration

#### 5.1 Configure Telegram Topic for URL Ingestion `[GUIDED]`

Link the knowledge base to the Telegram KB topic so that dropping a URL in the topic triggers automatic ingestion.

**Remote:**
```
openclaw config set knowledge-base.telegram-topic ${KB_TOPIC_ID}
```

Expected: Configuration saved. URLs posted to the KB topic in Telegram will now trigger ingestion.

If this fails: Check that OpenClaw is running (`openclaw health`). The `openclaw config` command requires the gateway to be active.

If already exists: `openclaw config set` overwrites the existing value. Safe to re-run with a different topic ID if needed.

### Phase 6: Optional Integrations

#### 6.1 Configure Slack Cross-Posting (Optional) `[HUMAN_INPUT]`

If the client wants ingested articles to also post to a Slack channel for team visibility, configure the cross-posting integration. **Skip this step if the client doesn't use Slack or doesn't want cross-posting.**

Ask the client: "Would you like ingested KB articles to be cross-posted to a Slack channel? If so, which channel?"

**Remote:**
```
openclaw config set knowledge-base.slack-crosspost true
openclaw config set knowledge-base.slack-channel ${SLACK_CHANNEL}
```

Expected: Configuration saved. Requires `SLACK_BOT_TOKEN` set in `~/.openclaw/.env`.

If this fails: Verify `SLACK_BOT_TOKEN` is configured and the bot has been invited to the target channel.

#### 6.2 Verify X/Twitter Thread Support (Optional) `[AUTO]`

If the client has X/Twitter API access and `bun` installed, verify the thread-following ingestion works.

**Remote:**
```
which bun && echo "bun available" || echo "bun not available -- X thread ingestion will be limited"
```

If `bun` is available, also check for the X API bearer token:

**Remote:**
```
grep X_BEARER_TOKEN ~/.openclaw/.env 2>/dev/null && echo "X API configured" || echo "X API not configured"
```

Expected: Both `bun` and `X_BEARER_TOKEN` available for full X/Twitter thread support. If either is missing, X ingestion falls back to basic URL scraping (single tweet only, no thread following).

If this fails: Not critical. X/Twitter thread support is optional. Note the limitation in the client profile.

## Verification

Run these checks to confirm the knowledge base is fully operational.

**Remote -- check database statistics:**
```
cd ${WORKSPACE}/skills/knowledge-base && node scripts/stats.js
```

Expected: Source count >= 1 (the test article), chunk count > 0, total word count > 0.

**Remote -- test semantic query:**
```
cd ${WORKSPACE}/skills/knowledge-base && node scripts/query.js 'test query'
```

Expected: Ranked results with relevance scores.

**Remote -- list recent sources:**
```
cd ${WORKSPACE}/skills/knowledge-base && node scripts/list.js --recent 5
```

Expected: Recently ingested sources with metadata (URL, title, type, tags, timestamp).

**Remote -- verify OpenClaw config:**
```
openclaw config get knowledge-base
```

Expected: Shows `telegram-topic` set to the KB topic ID. If Slack cross-posting was configured, shows `slack-crosspost: true` and `slack-channel`.

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Ingest fails on URL fetch | Content extraction CLI not installed or not on PATH | Install `summarize` (macOS) or alternative extraction tool (see Phase 2). Verify with `which summarize`. |
| Lock errors during ingestion | Stale lock file from crashed process | Check and remove `${WORKSPACE}/skills/knowledge-base/data/.ingest.lock` if no ingestion is actually running. |
| Twitter threads incomplete | Only first tweet captured | Verify `X_BEARER_TOKEN` is set in `~/.openclaw/.env`. Thread follower needs API access to resolve reply chains. Also verify `bun` is installed. |
| Paywalled content empty | No browser session available | Deploy the browser-control skill with a logged-in Chrome session. Ingestion falls back to browser automation for paywalled sites. |
| Embedding errors | `GEMINI_API_KEY` missing or expired | Verify `GEMINI_API_KEY` in `~/.openclaw/.env`. Test with a direct API call to the embedding endpoint. |
| YouTube transcript missing | Video has no captions | `summarize` CLI relies on YouTube captions. If the video has no captions, ingestion stores metadata only without transcript content. |
| Slack cross-post fails | Bot token invalid or channel not joined | Verify `SLACK_BOT_TOKEN` and ensure the bot has been invited to the target channel. |
| `npm install` fails | Node.js or npm not on PATH | Verify with `node --version && npm --version`. May need to source shell profile or refresh PATH. |
| Database locked | Concurrent ingestion processes | Only one ingestion should run at a time. Check for running node processes: `pgrep -f "ingest.js"`. |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Create directories | Execute silently | Execute silently | Confirm before creating |
| 1.2 Install npm dependencies | Execute silently | Confirm before install | Confirm each command |
| 2.1 Install content extraction CLI | Execute silently | Confirm tool choice (especially on non-macOS) | Confirm tool choice and each command |
| 3.2 Back up existing database | Execute silently | Confirm before backup | Confirm |
| 4.1 Test ingestion | Execute silently | Confirm before ingesting test article | Confirm |
| 4.2 Test query | Execute silently | Execute silently | Confirm |
| 5.1 Configure Telegram topic | Execute silently | Confirm topic ID | Confirm |
| 6.1 Slack cross-posting | Skip unless client profile says yes | Ask client | Ask client |
| 6.2 Verify X/Twitter support | Execute silently | Execute silently | Show results, confirm |

## Dependencies

- **Depends on:** `openclaw/install.md`, `deploy-messaging-setup.md`, `deploy-security-safety.md`
- **Required by:** `deploy-video-pipeline.md` (KB queries for video research), `deploy-advisory-council.md` (optional data source)
- **Enhanced by:** Browser-control skill (paywalled content), X API access (full thread following)
