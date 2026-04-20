---
name: "Deploy Video Research"
category: "integration"
subcategory: "content-gen"
third_party_service: "Telegram + Gemini API"
auth_type: "api-key"
openclaw_version_required: "2026.4.15+"
version_last_verified: "2026-04-20"
last_checked: "2026-04-20"
maintenance_status: "active"
model_env_var: "VIDEO_RESEARCH_MODEL"
clawhub_alternative: "none"
cost_tier: "api-usage"
privacy_tier: "cloud"
requires:
  - "Content pipeline database (deploy-content-pipeline.md)"
  - "Telegram Content Agency group configured"
  - "Gemini API key"
docs_urls:
  - "https://ai.google.dev/gemini-api/docs"
  - "https://core.telegram.org/bots/api"
---

# Deploy Video Research

## Compatibility
- **OpenClaw Version**: 2026.4.15+ (bumped 2026-04-20)
- **Status**: READY
- **Blocked Commands**: (none)
- **Model**: agent default (`agents.defaults.model` in `openclaw.json`). Override this skill's LLM routing via env var **`VIDEO_RESEARCH_MODEL`** (see `_authoring/_deploy-common.md` "Per-skill env-var overrides" for the pattern + examples). Applies to the trend-research + hook/caption/prompt draft pass.

## Purpose

Install the video research and content ideation system that operates through the **Content Agency** Telegram group. When the operator posts a video idea, the agent researches trends, drafts hooks/captions/prompts, and manages the full pipeline through approval gates — from idea to published TikTok video.

**When to use:** When the client wants an AI-powered content research and creation pipeline for TikTok, driven by Telegram conversations in the Content Agency group.

**What this skill does:**
1. Configures the Content Agency Telegram group as the content command center
2. Installs trend research capabilities (hashtag lookup, competitor analysis)
3. Creates the research-to-post workflow with approval gates
4. Sets up the agent's content pipeline behavior rules

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${CONTENT_DB}` | Content pipeline database | `${WORKSPACE}/data/content-pipeline.db` |
| `${CONTENT_AGENCY_CHAT_ID}` | Telegram group: Content Agency chat ID | `-1001234567890` |
| `${VIDEO_RESEARCH_TOPIC_ID}` | Telegram topic: Video Research Ideas thread ID | `142` |
| `${VIDEO_CREATION_TOPIC_ID}` | Telegram topic: Video Creation thread ID | `156` |
| `${VIDEO_APPROVAL_TOPIC_ID}` | Telegram topic: Video Approval thread ID | `170` |
| `${GEMINI_API_KEY}` | Client machine `.env` | `AIzaSy...` |
| `${TIKTOK_USERNAME}` | Client's TikTok account | `@relcontent` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`.

**Pre-flight checks for this skill:**

1. Content pipeline database exists:

**Remote:**
```
sqlite3 ${CONTENT_DB} "SELECT value FROM meta WHERE key='schema_version'" 2>/dev/null || echo 'NOT_DEPLOYED'
```

Expected: `1.0.0`. If `NOT_DEPLOYED`, run `deploy-content-pipeline.md` first.

2. Content Agency Telegram group is accessible:

The agent should already be admin of the Content Agency group with topic management. Verify by checking the agent can send messages to the group.

3. Video generation capability exists:

**Remote:**
```
test -d ${WORKSPACE}/skills/video-gen && echo 'EXISTS' || echo 'MISSING'
```

Expected: `EXISTS`. If missing, deploy `deploy-video-gen.md` for Veo 3, or ensure fal.ai is configured.

4. TikTok posting is deployed:

**Remote:**
```
test -f ${WORKSPACE}/skills/tiktok-post/post.py && echo 'EXISTS' || echo 'MISSING'
```

Expected: `EXISTS`. If missing, deploy `deploy-tiktok-posting.md`.

## Adaptation Points

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Trigger group | Content Agency (Telegram) | Client uses Slack or Discord instead |
| Research sources | Web search + hashtag lookup | Client has API keys for TickerTrends, Data365 |
| Video generation | Fal.ai Aurora / Veo 3 | Client prefers Kling, Runway, or local models |
| Posting platform | TikTok | Client also wants Instagram Reels, YouTube Shorts |
| Approval flow | 2-step (research → generate → post) | Client wants 3-step (add script approval) or 1-step (auto-post) |

## Prerequisites

- `deploy-content-pipeline.md` deployed (content database)
- `deploy-messaging-setup.md` deployed (Telegram configured)
- `deploy-tiktok-posting.md` deployed (posting capability)
- `deploy-video-gen.md` or fal.ai configured (video generation)
- Content Agency Telegram group created with agent as admin
- Topics exist: Video Research, Video Creation, Video Approval

## Steps

### Phase 1: Configure Content Agency Group

#### 1.1 Register group configuration `[GUIDED]`

Store the Content Agency Telegram group details in the agent's configuration so it knows which group and topics to monitor.

**Remote:**

Write group config to `${WORKSPACE}/config/content-agency.json`:

```json
{
  "telegram": {
    "chat_id": "${CONTENT_AGENCY_CHAT_ID}",
    "topics": {
      "video_research": {
        "thread_id": "${VIDEO_RESEARCH_TOPIC_ID}",
        "purpose": "the operator posts video ideas here. Agent researches and responds with findings."
      },
      "video_creation": {
        "thread_id": "${VIDEO_CREATION_TOPIC_ID}",
        "purpose": "Agent posts generated videos for the operator's review."
      },
      "video_approval": {
        "thread_id": "${VIDEO_APPROVAL_TOPIC_ID}",
        "purpose": "Final approval before posting to TikTok. Agent posts preview + caption for the operator to approve."
      }
    }
  },
  "tiktok": {
    "username": "${TIKTOK_USERNAME}",
    "cookies_path": "${WORKSPACE}/config/tiktok-cookies.txt"
  },
  "research": {
    "default_hashtag_count": 5,
    "trend_cache_hours": 48,
    "max_caption_length": 300,
    "video_specs": {
      "width": 1080,
      "height": 1920,
      "aspect_ratio": "9:16",
      "fps": 30,
      "duration_sweet_spot": "21-34 seconds"
    }
  },
  "approval_gates": {
    "research_to_generate": true,
    "generate_to_post": true
  }
}
```

Expected: Config file created with all topic IDs and settings.

If this fails: Ensure the directory exists. Verify topic IDs by checking Telegram group settings.

### Phase 2: Install Research Tools

#### 2.1 Install trend research script `[AUTO]`

Create the research script that gathers TikTok trend data for a given topic. Uses web search and optionally TickerTrends API if available.

**Remote:**

Write to `${WORKSPACE}/skills/video-research/research.py`:

```python
#!/usr/bin/env python3
"""Research TikTok trends for a video idea.

Usage:
  python3 research.py "<topic>" [--idea-id <id>] [--db <path>]

Outputs JSON with:
  - trending_hashtags: relevant hashtags with estimated reach
  - content_angles: suggested approaches/hooks
  - competitor_examples: similar successful videos
  - recommended_sounds: trending audio (if available)
  - prompt_suggestions: AI generation prompts for the topic
"""
import argparse
import json
import os
import sqlite3
import sys
from datetime import datetime, timedelta

def get_db(db_path):
    conn = sqlite3.connect(db_path)
    conn.row_factory = sqlite3.Row
    return conn

def cache_research(db, idea_id, source, query, data, ttl_hours=48):
    """Store research results with TTL."""
    expires = (datetime.utcnow() + timedelta(hours=ttl_hours)).isoformat()
    db.execute(
        "INSERT INTO trend_research (idea_id, source, query, data_json, expires_at) VALUES (?, ?, ?, ?, ?)",
        (idea_id, source, query, json.dumps(data), expires)
    )
    db.commit()

def cache_hashtags(db, hashtags):
    """Store trending hashtags with TTL."""
    expires = (datetime.utcnow() + timedelta(hours=48)).isoformat()
    for tag in hashtags:
        db.execute(
            "INSERT OR REPLACE INTO trending_hashtags (hashtag, platform, view_count, growth_rate, expires_at) VALUES (?, 'tiktok', ?, ?, ?)",
            (tag['hashtag'], tag.get('view_count'), tag.get('growth_rate'), expires)
        )
    db.commit()

def get_cached_research(db, query, max_age_hours=48):
    """Check for fresh cached research."""
    cutoff = (datetime.utcnow() - timedelta(hours=max_age_hours)).isoformat()
    row = db.execute(
        "SELECT data_json FROM trend_research WHERE query=? AND created_at > ? ORDER BY created_at DESC LIMIT 1",
        (query, cutoff)
    ).fetchone()
    return json.loads(row['data_json']) if row else None

def research_topic(topic, db, idea_id=None):
    """
    Main research function. The agent should call this and then
    use its own AI capabilities to:
    1. Web search for "{topic} TikTok trending"
    2. Search TikTok Creative Center (if accessible)
    3. Analyze competitor videos on the topic
    4. Generate hook and caption suggestions

    This script provides the framework — the agent fills in the
    research using its tools (web search, browsing, AI analysis).
    """
    # Check cache first
    cached = get_cached_research(db, topic)
    if cached:
        print(json.dumps({"status": "cached", "data": cached}))
        return cached

    # Skeleton for the agent to populate
    research = {
        "topic": topic,
        "researched_at": datetime.utcnow().isoformat(),
        "trending_hashtags": [],
        "content_angles": [],
        "competitor_examples": [],
        "recommended_sounds": [],
        "prompt_suggestions": {
            "fal_aurora": "",
            "veo3": "",
            "kling": ""
        },
        "caption_drafts": [],
        "hooks": [],
        "recommended_duration": "21-34 seconds",
        "recommended_posting_time": None
    }

    # Get best posting window
    window = db.execute(
        "SELECT day_of_week, hour_utc, score FROM posting_windows WHERE platform='tiktok' ORDER BY score DESC LIMIT 1"
    ).fetchone()
    if window:
        days = ['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun']
        research["recommended_posting_time"] = f"{days[window['day_of_week']]} {window['hour_utc']}:00 UTC (score: {window['score']})"

    # Cache and return
    cache_research(db, idea_id, 'web_search', topic, research)

    if idea_id:
        db.execute(
            "UPDATE content_ideas SET status='researched', research_summary=?, updated_at=datetime('now') WHERE id=?",
            (json.dumps(research, indent=2)[:2000], idea_id)
        )
        db.execute(
            "INSERT INTO status_log (idea_id, from_status, to_status, changed_by, reason) VALUES (?, 'idea', 'researched', 'rel', ?)",
            (idea_id, f"Trend research completed for: {topic}")
        )
        db.commit()

    print(json.dumps({"status": "researched", "data": research}, indent=2))
    return research

def main():
    parser = argparse.ArgumentParser(description='Research TikTok trends for a topic')
    parser.add_argument('topic', help='The video topic to research')
    parser.add_argument('--idea-id', type=int, help='Content idea ID in the pipeline', default=None)
    parser.add_argument('--db', default=None, help='Path to content-pipeline.db')
    args = parser.parse_args()

    workspace = os.environ.get('WORKSPACE', os.path.expanduser('~/clawd'))
    db_path = args.db or os.path.join(workspace, 'data', 'content-pipeline.db')

    if not os.path.exists(db_path):
        print(f"ERROR: Database not found: {db_path}", file=sys.stderr)
        sys.exit(1)

    db = get_db(db_path)
    try:
        research_topic(args.topic, db, args.idea_id)
    finally:
        db.close()

if __name__ == '__main__':
    main()
```

Expected: Script created at `${WORKSPACE}/skills/video-research/research.py`.

**Remote:**
```
chmod +x ${WORKSPACE}/skills/video-research/research.py
```

### Phase 3: Configure Agent Behavior

#### 3.1 Write content pipeline behavior rules `[GUIDED]`

Define the agent's behavioral rules for the content pipeline. These rules tell the agent how to handle messages in the Content Agency group and manage the workflow.

**Remote:**

Write to `${WORKSPACE}/config/content-pipeline-rules.md`:

```markdown
# Content Pipeline Rules

## Listening

You monitor the **Content Agency** Telegram group. When the operator posts in the **Video Research** topic, treat it as a new video idea unless it's clearly a question or comment on existing work.

## Workflow: Idea → Posted Video

### Step 1: Capture Idea
When the operator posts a video idea in the Video Research topic:
1. Create a new row in content_ideas (source='telegram', capture the message ID)
2. Acknowledge in the thread: "Got it — researching [topic] now."
3. Move to Step 2 immediately.

### Step 2: Research
1. Web search for "[topic] TikTok trending" and "[topic] TikTok viral"
2. Find 5-8 relevant trending hashtags with view counts
3. Identify 2-3 successful competitor videos on this topic
4. Note any trending sounds that fit
5. Cache all research in the trend_research table
6. Update idea status to 'researched'

### Step 3: Present Research (APPROVAL GATE 1)
Post a research summary in the Video Research topic:

```
📊 Research: [Topic]

🔥 Trending hashtags:
#hashtag1 (X views) #hashtag2 (Y views) ...

🎯 Content angles:
1. [Angle 1 — why it works]
2. [Angle 2 — why it works]
3. [Angle 3 — why it works]

🎬 Competitor examples:
- [Brief description of successful video 1]
- [Brief description of successful video 2]

🎵 Trending sounds: [if applicable]

📝 My recommendation: [which angle + why]

Reply "go" to approve, or suggest changes.
```

**Wait for the operator's approval before proceeding.** Do not generate anything without explicit approval.

### Step 4: Script & Prompt (after approval)
1. Generate 2-3 hook variants (A/B/C) — store in creative_drafts
2. Generate 2 caption options with hashtags — store in creative_drafts
3. Create the AI generation prompt for the chosen model — store in generation_prompts
4. Post hooks and captions in Video Research topic for the operator to pick
5. Update idea status to 'scripted' after the operator selects

### Step 5: Generate Video
1. Send the prompt to the chosen AI model (Fal.ai Aurora, Veo 3, or Kling)
2. Update idea status to 'generating'
3. When complete, update to 'generated'
4. Post the video to the **Video Creation** topic with the selected caption
5. Ask the operator: "Happy with this? Reply 'post' to publish, or 'redo' for another take."

### Step 6: Multi-Platform Post (APPROVAL GATE 2)
After the operator approves in Video Creation:
1. Post preview to **Video Approval** topic with final caption + hashtags
2. Show the posting plan: "Publishing to: TikTok + YouTube Shorts. Reply 'post' to confirm."
3. Wait for the operator's confirmation
4. Run the compliance gate (metadata strip, frequency check, ban check)
5. Post to **all configured platforms** in sequence:
   a. **TikTok** — via tiktok-uploader (strip metadata first)
   b. **YouTube Shorts** — via YouTube Data API (vertical format auto-detected, AI content flag set)
   c. **YouTube long-form** — only if the operator specifically requests a longer version
6. Update post status for each platform in database
7. Confirm in Video Approval: "Posted! TikTok: [URL] | YouTube: [URL]"
8. Update idea status to 'posted'

### Step 7: Track Performance
After posting, pull metrics at:
- 1 hour
- 6 hours
- 24 hours
- 48 hours
- 7 days

Store each snapshot in post_metrics **per platform**. Compare cross-platform performance to learn which content works better where.

## Rules

1. **Never post without explicit approval.** Both approval gates are mandatory.
2. **Always use the Content Agency group** — never DM the operator about content pipeline stuff.
3. **Use the correct topic** — research in Video Research, previews in Video Creation, final approval in Video Approval.
4. **Cache research aggressively** — don't re-research the same topic within 48 hours.
5. **Track everything in the database** — every status change gets logged.
6. **Keep TikTok captions under 300 characters** — TikTok sweet spot.
7. **Always include 3-5 hashtags** — mix of trending (high reach) and niche (high relevance).
8. **Video specs: 1080x1920, 9:16, 30fps, 21-34 seconds** (works for both TikTok and YouTube Shorts).
9. **Hashtags must be followed by a space** in TikTok caption (tiktok-uploader requirement).
10. **Always set AI disclosure flags** — TikTok AIGC label ON, YouTube `containsSyntheticMedia: true`.
11. **Always strip metadata before TikTok upload** — run compliance gate. YouTube doesn't need this (official API).
12. **Post to all platforms from one approval** — the operator approves once, agent posts to TikTok + YouTube Shorts.
13. **YouTube descriptions can be longer** — include full description with links, longer hashtag list, and cross-platform references.
```

Expected: Rules file created at `${WORKSPACE}/config/content-pipeline-rules.md`.

#### 3.2 Register behavior rules with agent `[GUIDED]`

Add the content pipeline rules to the agent's system prompt or behavior configuration so it loads them on startup.

The agent's main config (typically `${WORKSPACE}/config/system-prompt.md` or equivalent) should reference:
```
Load and follow the rules in config/content-pipeline-rules.md for all Content Agency group interactions.
```

The operator should add this reference to whatever mechanism the agent uses to load behavior rules on startup. This varies per agent setup.

Expected: Agent loads content pipeline rules when processing Content Agency group messages.

### Phase 4: End-to-End Test

#### 4.1 Simulate the full workflow `[GUIDED]`

Test the pipeline with a real idea. the operator posts a test idea in the Video Research topic.

1. Agent captures it → creates content_ideas row
2. Agent researches → updates status, caches results
3. Agent posts research summary → waits for approval
4. the operator approves → agent scripts hooks/captions
5. Agent generates video → posts in Video Creation
6. the operator approves → agent posts in Video Approval
7. the operator confirms → agent posts to TikTok (or dry-run)

Verify at each step:
```
sqlite3 ${CONTENT_DB} "SELECT id, title, status FROM content_ideas ORDER BY id DESC LIMIT 1"
```

Expected: Status progresses through: idea → researched → approved → scripted → generating → generated → review → posted.

## Verification

1. **Config exists:**
   ```
   cat ${WORKSPACE}/config/content-agency.json | python3 -c "import sys,json; d=json.load(sys.stdin); print(len(d['telegram']['topics']), 'topics configured')"
   ```

2. **Research script runs:**
   ```
   python3 ${WORKSPACE}/skills/video-research/research.py "test topic" --db ${CONTENT_DB}
   ```

3. **Database tracking works:**
   ```
   sqlite3 ${CONTENT_DB} "SELECT count(*) FROM trend_research"
   ```

4. **Rules file loaded:**
   ```
   test -f ${WORKSPACE}/config/content-pipeline-rules.md && echo 'EXISTS' || echo 'MISSING'
   ```

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Register group config | Execute | Confirm | Confirm |
| 2.1 Install research script | Execute | Execute | Confirm |
| 3.1 Write behavior rules | Execute | Confirm | Confirm |
| 3.2 Register with agent | SKIP (needs operator) | Confirm | Confirm |
| 4.1 End-to-end test | SKIP | Guide | Guide |

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Agent doesn't respond to ideas | Group config not loaded or wrong chat_id | Verify `content-agency.json` chat_id matches the actual group |
| Research comes back empty | Web search not available or rate limited | Check agent's web search capability; try manual research |
| Video generation fails | Missing API key or wrong model | Check `GEMINI_API_KEY` or fal.ai token in `.env` |
| TikTok post fails | Cookies expired | Re-export cookies (see `deploy-tiktok-posting.md`) |
| Status not updating | Database path mismatch | Check `${CONTENT_DB}` resolves correctly in all scripts |

## Dependencies

- **Depends on:** `deploy-content-pipeline.md` (database), `deploy-tiktok-posting.md` (posting), `deploy-video-gen.md` (generation), `deploy-messaging-setup.md` (Telegram)
- **Required by:** (none — this is the top-level orchestration skill)
