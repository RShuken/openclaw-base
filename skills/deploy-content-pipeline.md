---
name: "Deploy Content Pipeline"
category: "integration"
subcategory: "content-gen"
third_party_service: "SQLite (local) + Telegram"
auth_type: "api-key"
openclaw_version_required: "2026.4.15+"
version_last_verified: "2026-04-20"
last_checked: "2026-04-20"
maintenance_status: "active"
model_env_var: "CONTENT_PIPELINE_MODEL"
clawhub_alternative: "none"
cost_tier: "free"
privacy_tier: "hybrid"
requires:
  - "SQLite"
  - "Telegram Content Agency group configured"
docs_urls:
  - "https://www.sqlite.org/docs.html"
  - "https://core.telegram.org/bots/api"
---

# Deploy Content Pipeline

## Compatibility
- **OpenClaw Version**: 2026.4.15+ (bumped 2026-04-20)
- **Status**: READY
- **Blocked Commands**: (none)
- **Model**: this skill itself makes no LLM calls (it's pure DB + schema setup). Downstream skills that consume this DB (`deploy-video-research.md`, `deploy-tiktok-posting.md`, Telegram idea-capture flows) route content-generation LLM calls through env var **`CONTENT_PIPELINE_MODEL`** (see `_authoring/_deploy-common.md` "Per-skill env-var overrides" for the pattern + examples). Set this once in `~/.openclaw/.env` and all pipeline scripts will pick it up.

## Purpose

Install a content management database for the TikTok/social media content pipeline. This is a **separate system from the personal CRM** — it tracks content ideas, trend research, video assets, publishing schedule, and performance metrics. The database powers the full workflow from idea capture in Telegram through posting and analytics.

**When to use:** Before deploying `deploy-tiktok-posting.md` or `deploy-video-research.md`. This is the data foundation for all content operations.

**What this skill does:**
1. Creates the content pipeline SQLite database (14 tables)
2. Seeds default posting windows (optimal TikTok times)
3. Creates utility scripts for common queries
4. Verifies the database is separate from the CRM

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${CONTENT_DB}` | Derived: `${WORKSPACE}/data/content-pipeline.db` | `~/clawd/data/content-pipeline.db` |
| `${CONTENT_AGENCY_CHAT_ID}` | Telegram group: Content Agency chat ID | `-1001234567890` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`.

**Pre-flight checks for this skill:**

1. Check if content pipeline database already exists:

**Remote:**
```
test -f ${WORKSPACE}/data/content-pipeline.db && echo 'EXISTS' || echo 'NOT_DEPLOYED'
```

Expected: `NOT_DEPLOYED` for fresh installs.

If already exists: Check schema version in `meta` table. If outdated, run migrations. If current, skip.

2. Confirm CRM database is at a DIFFERENT path:

**Remote:**
```
ls ${WORKSPACE}/crm/data/contacts.db 2>/dev/null && echo 'CRM_FOUND' || echo 'NO_CRM'
```

Expected: `CRM_FOUND` (confirms CRM exists separately) or `NO_CRM` (CRM not deployed yet, which is fine).

**Critical:** The content pipeline database MUST be at `${WORKSPACE}/data/content-pipeline.db`, never inside `${WORKSPACE}/crm/`. These systems must not cross-contaminate.

## Adaptation Points

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Database path | `${WORKSPACE}/data/content-pipeline.db` | Client wants content data in a different location |
| Posting windows | TikTok defaults (Tue-Thu, 10am-2pm) | Client has analytics showing different optimal times |
| Platforms | TikTok only | Client wants Instagram Reels, YouTube Shorts columns |

## Prerequisites

- OpenClaw installed and running
- `${WORKSPACE}/data/` directory exists
- SQLite3 available on the client machine

## Steps

### Phase 1: Create Database

#### 1.1 Create the content pipeline database `[AUTO]`

Create the SQLite database with all 14 tables. This schema tracks the full content lifecycle from idea through performance analytics.

**Remote:**

Execute the following SQL via `sqlite3 ${CONTENT_DB}`:

```sql
-- Content Pipeline Database
-- Separate from CRM (contacts.db) — content operations only

PRAGMA journal_mode=WAL;
PRAGMA foreign_keys=ON;

-- Tags for organizing content
CREATE TABLE IF NOT EXISTS tags (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL UNIQUE,
    category TEXT NOT NULL CHECK(category IN ('theme', 'series', 'audience', 'format', 'platform')),
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Central table: every video starts as an idea
CREATE TABLE IF NOT EXISTS content_ideas (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    description TEXT,
    source TEXT NOT NULL DEFAULT 'telegram' CHECK(source IN ('telegram', 'trend_research', 'advisory_council', 'manual', 'competitor')),
    status TEXT NOT NULL DEFAULT 'idea' CHECK(status IN ('idea', 'researched', 'approved', 'scripted', 'generating', 'generated', 'review', 'scheduled', 'posted', 'failed', 'archived')),
    priority INTEGER DEFAULT 0,
    telegram_chat_id TEXT,
    telegram_message_id TEXT,
    telegram_thread_id TEXT,
    research_summary TEXT,
    approved_at TEXT,
    approved_by TEXT DEFAULT 'operator',
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Many-to-many: ideas <-> tags
CREATE TABLE IF NOT EXISTS content_idea_tags (
    idea_id INTEGER NOT NULL REFERENCES content_ideas(id) ON DELETE CASCADE,
    tag_id INTEGER NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (idea_id, tag_id)
);

-- Cached trend research (TTL-based)
CREATE TABLE IF NOT EXISTS trend_research (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    idea_id INTEGER REFERENCES content_ideas(id) ON DELETE SET NULL,
    source TEXT NOT NULL CHECK(source IN ('tiktok_creative_center', 'tickertrends', 'competitor', 'web_search', 'manual')),
    query TEXT NOT NULL,
    data_json TEXT NOT NULL,
    expires_at TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Standalone hashtag trends (queried independently)
CREATE TABLE IF NOT EXISTS trending_hashtags (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    hashtag TEXT NOT NULL,
    platform TEXT NOT NULL DEFAULT 'tiktok',
    view_count INTEGER,
    growth_rate REAL,
    data_json TEXT,
    expires_at TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    UNIQUE(hashtag, platform)
);

-- Multiple drafts per idea: hooks, captions, prompts
CREATE TABLE IF NOT EXISTS creative_drafts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    idea_id INTEGER NOT NULL REFERENCES content_ideas(id) ON DELETE CASCADE,
    draft_type TEXT NOT NULL CHECK(draft_type IN ('hook', 'caption', 'script', 'cta', 'prompt')),
    variant TEXT DEFAULT 'A' CHECK(variant IN ('A', 'B', 'C', 'D')),
    content TEXT NOT NULL,
    is_selected INTEGER DEFAULT 0,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- AI generation prompts (versioned)
CREATE TABLE IF NOT EXISTS generation_prompts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    idea_id INTEGER NOT NULL REFERENCES content_ideas(id) ON DELETE CASCADE,
    provider TEXT NOT NULL CHECK(provider IN ('fal_aurora', 'veo3', 'kling', 'gemini_image', 'other')),
    prompt_text TEXT NOT NULL,
    negative_prompt TEXT,
    params_json TEXT,
    version INTEGER DEFAULT 1,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Generated video/image assets
CREATE TABLE IF NOT EXISTS video_assets (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    idea_id INTEGER NOT NULL REFERENCES content_ideas(id) ON DELETE CASCADE,
    prompt_id INTEGER REFERENCES generation_prompts(id) ON DELETE SET NULL,
    asset_type TEXT NOT NULL DEFAULT 'video' CHECK(asset_type IN ('video', 'thumbnail', 'image', 'source_clip')),
    file_path TEXT,
    r2_bucket TEXT,
    r2_key TEXT,
    cdn_url TEXT,
    duration_seconds REAL,
    width INTEGER,
    height INTEGER,
    file_size_bytes INTEGER,
    generation_status TEXT DEFAULT 'pending' CHECK(generation_status IN ('pending', 'processing', 'completed', 'failed')),
    generation_cost_cents INTEGER,
    generation_started_at TEXT,
    generation_completed_at TEXT,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Publishing records
CREATE TABLE IF NOT EXISTS posts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    idea_id INTEGER NOT NULL REFERENCES content_ideas(id) ON DELETE CASCADE,
    video_asset_id INTEGER REFERENCES video_assets(id) ON DELETE SET NULL,
    platform TEXT NOT NULL DEFAULT 'tiktok' CHECK(platform IN ('tiktok', 'instagram_reels', 'youtube_shorts', 'x')),
    status TEXT NOT NULL DEFAULT 'draft' CHECK(status IN ('draft', 'scheduled', 'posting', 'posted', 'failed')),
    caption TEXT,
    hashtags_json TEXT,
    scheduled_at TEXT,
    published_at TEXT,
    platform_post_id TEXT,
    platform_url TEXT,
    approval_telegram_message_id TEXT,
    error_message TEXT,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Optimal posting times (learned over time)
CREATE TABLE IF NOT EXISTS posting_windows (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    platform TEXT NOT NULL DEFAULT 'tiktok',
    day_of_week INTEGER NOT NULL CHECK(day_of_week BETWEEN 0 AND 6),
    hour_utc INTEGER NOT NULL CHECK(hour_utc BETWEEN 0 AND 23),
    score REAL NOT NULL DEFAULT 0.5 CHECK(score BETWEEN 0.0 AND 1.0),
    sample_count INTEGER DEFAULT 0,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    UNIQUE(platform, day_of_week, hour_utc)
);

-- Time-series performance metrics
CREATE TABLE IF NOT EXISTS post_metrics (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    post_id INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    hours_since_post INTEGER NOT NULL,
    views INTEGER DEFAULT 0,
    likes INTEGER DEFAULT 0,
    comments INTEGER DEFAULT 0,
    shares INTEGER DEFAULT 0,
    saves INTEGER DEFAULT 0,
    avg_watch_seconds REAL,
    completion_rate REAL,
    profile_visits INTEGER DEFAULT 0,
    follower_gain INTEGER DEFAULT 0,
    collected_at TEXT NOT NULL DEFAULT (datetime('now')),
    UNIQUE(post_id, hours_since_post)
);

-- Content calendar
CREATE TABLE IF NOT EXISTS calendar_slots (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    slot_date TEXT NOT NULL,
    slot_time TEXT,
    idea_id INTEGER REFERENCES content_ideas(id) ON DELETE SET NULL,
    platform TEXT DEFAULT 'tiktok',
    theme TEXT,
    series TEXT,
    status TEXT DEFAULT 'open' CHECK(status IN ('open', 'planned', 'filled', 'skipped')),
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Audit trail for status changes
CREATE TABLE IF NOT EXISTS status_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    idea_id INTEGER NOT NULL REFERENCES content_ideas(id) ON DELETE CASCADE,
    from_status TEXT,
    to_status TEXT NOT NULL,
    changed_by TEXT DEFAULT 'agent' CHECK(changed_by IN ('agent', 'operator', 'system')),
    reason TEXT,
    telegram_message_id TEXT,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Key-value config store
CREATE TABLE IF NOT EXISTS meta (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL,
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Indexes for common queries
CREATE INDEX IF NOT EXISTS idx_ideas_status ON content_ideas(status);
CREATE INDEX IF NOT EXISTS idx_ideas_telegram ON content_ideas(telegram_chat_id, telegram_message_id);
CREATE INDEX IF NOT EXISTS idx_ideas_priority ON content_ideas(priority DESC) WHERE status NOT IN ('posted', 'archived');
CREATE INDEX IF NOT EXISTS idx_posts_scheduled ON posts(scheduled_at) WHERE status = 'scheduled';
CREATE INDEX IF NOT EXISTS idx_posts_platform ON posts(platform, status);
CREATE INDEX IF NOT EXISTS idx_metrics_post ON post_metrics(post_id, hours_since_post);
CREATE INDEX IF NOT EXISTS idx_calendar_date ON calendar_slots(slot_date);
CREATE INDEX IF NOT EXISTS idx_assets_status ON video_assets(generation_status) WHERE generation_status IN ('pending', 'processing');
CREATE INDEX IF NOT EXISTS idx_hashtags_expires ON trending_hashtags(expires_at);
CREATE INDEX IF NOT EXISTS idx_research_expires ON trend_research(expires_at);
CREATE INDEX IF NOT EXISTS idx_status_log_idea ON status_log(idea_id, created_at);

-- Schema version
INSERT OR REPLACE INTO meta (key, value, updated_at) VALUES ('schema_version', '1.0.0', datetime('now'));
INSERT OR REPLACE INTO meta (key, value, updated_at) VALUES ('created_by', 'deploy-content-pipeline', datetime('now'));
```

Expected: Database file created at `${CONTENT_DB}` with 14 tables.

If this fails: Check that `${WORKSPACE}/data/` directory exists. Check SQLite3 is installed.

#### 1.2 Seed default posting windows `[AUTO]`

Seed the posting_windows table with TikTok best-practice times. These will be refined as performance data accumulates.

**Remote:**

Execute via `sqlite3 ${CONTENT_DB}`:

```sql
-- TikTok optimal posting times (UTC)
-- Based on general best practices: Tue-Thu 10am-2pm, with secondary windows
-- Scores: 0.9 = prime time, 0.7 = good, 0.5 = average

-- Tuesday (2)
INSERT OR IGNORE INTO posting_windows (platform, day_of_week, hour_utc, score) VALUES ('tiktok', 2, 14, 0.9);
INSERT OR IGNORE INTO posting_windows (platform, day_of_week, hour_utc, score) VALUES ('tiktok', 2, 15, 0.85);
INSERT OR IGNORE INTO posting_windows (platform, day_of_week, hour_utc, score) VALUES ('tiktok', 2, 17, 0.8);
INSERT OR IGNORE INTO posting_windows (platform, day_of_week, hour_utc, score) VALUES ('tiktok', 2, 18, 0.75);

-- Wednesday (3)
INSERT OR IGNORE INTO posting_windows (platform, day_of_week, hour_utc, score) VALUES ('tiktok', 3, 15, 0.9);
INSERT OR IGNORE INTO posting_windows (platform, day_of_week, hour_utc, score) VALUES ('tiktok', 3, 16, 0.85);
INSERT OR IGNORE INTO posting_windows (platform, day_of_week, hour_utc, score) VALUES ('tiktok', 3, 19, 0.8);

-- Thursday (4)
INSERT OR IGNORE INTO posting_windows (platform, day_of_week, hour_utc, score) VALUES ('tiktok', 4, 17, 0.9);
INSERT OR IGNORE INTO posting_windows (platform, day_of_week, hour_utc, score) VALUES ('tiktok', 4, 18, 0.85);
INSERT OR IGNORE INTO posting_windows (platform, day_of_week, hour_utc, score) VALUES ('tiktok', 4, 14, 0.8);

-- Friday (5)
INSERT OR IGNORE INTO posting_windows (platform, day_of_week, hour_utc, score) VALUES ('tiktok', 5, 15, 0.8);
INSERT OR IGNORE INTO posting_windows (platform, day_of_week, hour_utc, score) VALUES ('tiktok', 5, 16, 0.75);

-- Saturday (6) and Sunday (0) — lower scores
INSERT OR IGNORE INTO posting_windows (platform, day_of_week, hour_utc, score) VALUES ('tiktok', 6, 16, 0.7);
INSERT OR IGNORE INTO posting_windows (platform, day_of_week, hour_utc, score) VALUES ('tiktok', 0, 17, 0.7);

-- Monday (1)
INSERT OR IGNORE INTO posting_windows (platform, day_of_week, hour_utc, score) VALUES ('tiktok', 1, 16, 0.75);
INSERT OR IGNORE INTO posting_windows (platform, day_of_week, hour_utc, score) VALUES ('tiktok', 1, 17, 0.7);
```

Expected: 16 rows in posting_windows table.

### Phase 2: Create Utility Scripts

#### 2.1 Write pipeline status script `[AUTO]`

Create a quick-status script the agent uses to report pipeline health in Telegram.

**Remote:**

Write the following to `${WORKSPACE}/skills/content-pipeline/status.sh`:

```bash
#!/bin/bash
# Content pipeline status summary
DB="${1:-${WORKSPACE}/data/content-pipeline.db}"

echo "=== Content Pipeline Status ==="
echo ""

sqlite3 "$DB" "
SELECT '📊 Ideas by status:';
SELECT '  ' || status || ': ' || count(*) FROM content_ideas GROUP BY status ORDER BY count(*) DESC;
SELECT '';
SELECT '🎬 Video assets:';
SELECT '  ' || generation_status || ': ' || count(*) FROM video_assets GROUP BY generation_status;
SELECT '';
SELECT '📤 Posts:';
SELECT '  ' || platform || ' ' || status || ': ' || count(*) FROM posts GROUP BY platform, status;
SELECT '';
SELECT '📅 Next scheduled:';
SELECT '  ' || scheduled_at || ' - ' || caption FROM posts WHERE status='scheduled' ORDER BY scheduled_at LIMIT 3;
SELECT '';
SELECT '🔥 Top performing (7d):';
SELECT '  ' || p.platform_url || ' — ' || pm.views || ' views'
FROM post_metrics pm JOIN posts p ON p.id=pm.post_id
WHERE pm.hours_since_post=168 ORDER BY pm.views DESC LIMIT 3;
"
```

Expected: Script created, executable after `chmod +x`.

**Remote:**
```
chmod +x ${WORKSPACE}/skills/content-pipeline/status.sh
```

## Verification

1. **Database exists and has correct table count:**
   ```
   sqlite3 ${CONTENT_DB} "SELECT count(*) FROM sqlite_master WHERE type='table'"
   ```
   Expected: `14`

2. **Schema version set:**
   ```
   sqlite3 ${CONTENT_DB} "SELECT value FROM meta WHERE key='schema_version'"
   ```
   Expected: `1.0.0`

3. **Posting windows seeded:**
   ```
   sqlite3 ${CONTENT_DB} "SELECT count(*) FROM posting_windows"
   ```
   Expected: `16`

4. **CRM is separate:**
   ```
   sqlite3 ${CONTENT_DB} "SELECT count(*) FROM sqlite_master WHERE name='contacts'"
   ```
   Expected: `0` (no contacts table — this is NOT the CRM)

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Create database | Execute | Confirm | Confirm |
| 1.2 Seed posting windows | Execute | Confirm | Confirm |
| 2.1 Status script | Execute | Execute | Confirm |

## Dependencies

- **Depends on:** OpenClaw installed, SQLite3 available
- **Required by:** `deploy-tiktok-posting.md`, `deploy-video-research.md`
