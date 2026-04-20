# Deploy Knowledge Base (RAG)

## Purpose

Install a personal knowledge base with RAG (Retrieval-Augmented Generation). Ingest URLs via Telegram -- articles, YouTube transcripts, X/Twitter threads, PDFs. Supports semantic search with time-aware and source-weighted ranking.

**When to use:** When the user wants to save and query reference material from the web.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "ls ${WORKSPACE}/skills/knowledge-base/data/knowledge.db 2>/dev/null"
cmd --session <id> --command "which summarize 2>/dev/null"
cmd --session <id> --command "which bun 2>/dev/null"
```
- Is there an existing knowledge base? If so, this is an upgrade.
- Is the `summarize` CLI installed? (Required for URL ingestion)
- Is `bun` available? (Needed for X/Twitter thread ingestion via x-research-v2)
- Does the client want Slack cross-posting? (Optional)

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Alternative | When to Adapt |
|-----------|---------|-------------|---------------|
| Ingestion trigger | Telegram KB topic | Discord channel, Slack channel, CLI-only | Client's messaging platform |
| Embedding provider | Google `gemini-embedding-001` | OpenAI `text-embedding-3-small` | Client preference or API availability |
| X/Twitter ingestion | x-research-v2 via `bun` + X API | Disabled, or Grok fallback | Client doesn't have X API access or bun |
| Paywalled content | Browser automation via Chrome session | Skip paywalled, or use Firecrawl/Apify | Client doesn't want browser automation |
| Slack cross-post | Optional to #ai_trends | Different channel, disabled | Client's Slack setup |
| Database path | `${WORKSPACE}/skills/knowledge-base/data/knowledge.db` | Custom path | Non-standard workspace |
| Summarize CLI | `brew install steipete/tap/summarize` | npm install, or skip (reduces functionality) | Non-macOS, or Homebrew not available |

## Prerequisites

- `deploy-messaging-setup.md` completed (Telegram KB topic)
- `deploy-security-safety.md` completed (content sanitizer for web ingestion)
- Node.js + npm available
- `summarize` CLI installed: `brew install steipete/tap/summarize`
- Optional: `bun` runtime for X/Twitter research

## What Gets Installed

### SQLite Database (`~/clawd/skills/knowledge-base/data/knowledge.db`)

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
| ingest-and-crosspost.js | Ingest + optional Slack #ai_trends cross-post |
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

### Telegram Integration

- KB topic (ID: 1173) -- drop URLs to ingest
- Optional Slack cross-post to #ai_trends channel

## Steps

### 1. Create KB Skill Directory

```
cmd --session <id> --command "mkdir -p ~/clawd/skills/knowledge-base/data ~/clawd/skills/knowledge-base/scripts ~/clawd/skills/knowledge-base/src"
```

### 2. Install Dependencies

```
cmd --session <id> --command "cd ~/clawd/skills/knowledge-base && npm install"
```

### 3. Install Summarize CLI

The summarize CLI handles URL content extraction and YouTube transcript pulling.

```
cmd --session <id> --command "brew install steipete/tap/summarize"
```

Verify it is available:

```
cmd --session <id> --command "which summarize && summarize --version"
```

### 4. Initialize Database

The database is auto-created on first ingest, but you can verify the path exists:

```
cmd --session <id> --command "ls -la ~/clawd/skills/knowledge-base/data/"
```

### 5. Test Ingestion

Ingest a test article to confirm the full pipeline works (fetch, extract, chunk, embed, store).

```
cmd --session <id> --command "cd ~/clawd/skills/knowledge-base && node scripts/ingest.js 'https://example.com/article' --tags test"
```

### 6. Test Query

Run a semantic search query against the ingested content.

```
cmd --session <id> --command "cd ~/clawd/skills/knowledge-base && node scripts/query.js 'What was that article about?'"
```

### 7. Configure Telegram Topic for URL Ingestion

Set up topic 1173 so dropping a URL in Telegram triggers ingestion.

```
cmd --session <id> --command "openclaw config set knowledge-base.telegram-topic 1173"
```

### 8. Optional: Configure Slack Cross-Posting

If you want ingested articles to also post to a Slack channel for team visibility:

```
cmd --session <id> --command "openclaw config set knowledge-base.slack-crosspost true"
cmd --session <id> --command "openclaw config set knowledge-base.slack-channel ai_trends"
```

Requires `SLACK_BOT_TOKEN` set in `~/.openclaw/.env`.

## Verification

Run these commands to confirm the knowledge base is operational:

```
cmd --session <id> --command "cd ~/clawd/skills/knowledge-base && node scripts/stats.js"
cmd --session <id> --command "cd ~/clawd/skills/knowledge-base && node scripts/query.js 'test query'"
cmd --session <id> --command "cd ~/clawd/skills/knowledge-base && node scripts/list.js --recent 5"
```

Expected output:
- Stats should show source count, chunk count, total word count
- Query should return ranked results with relevance scores
- List should show recently ingested sources with metadata

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Ingest fails on URL fetch | summarize CLI not installed or not in PATH | Run `brew install steipete/tap/summarize` and verify with `which summarize`. |
| Lock errors during ingestion | Stale lock file from crashed process | Check and remove `~/clawd/skills/knowledge-base/data/.ingest.lock` if the process is not actually running. |
| Twitter threads incomplete | Only first tweet captured | Verify `X_BEARER_TOKEN` is set in `~/.openclaw/.env` for API access. The thread follower needs API access to resolve reply chains. |
| Paywalled content empty | No browser session available | Deploy the browser-control skill with a logged-in Chrome session. Ingestion falls back to browser automation for paywalled sites. |
| Embedding errors | GEMINI_API_KEY missing or expired | Verify `GEMINI_API_KEY` in `~/.openclaw/.env`. Test with a direct API call to the embedding endpoint. |
| YouTube transcript missing | Video has no captions | summarize CLI relies on YouTube captions. If the video has no captions, ingestion will store metadata only without transcript content. |
| Slack cross-post fails | Bot token invalid or channel not joined | Verify `SLACK_BOT_TOKEN` and ensure the bot has been invited to the target channel. |

## Dependencies

- **Requires:** `deploy-messaging-setup.md`, `deploy-security-safety.md`
- **Required by:** `deploy-video-pipeline.md`, `deploy-advisory-council.md` (optional)
