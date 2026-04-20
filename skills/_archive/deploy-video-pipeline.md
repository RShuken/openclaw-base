# Deploy Video Idea Pipeline

## Purpose

Install an automated video idea research pipeline triggered by Slack mentions. Reads full thread, runs X/Twitter research, queries knowledge base, creates structured Asana card with research findings. Includes semantic dedup to prevent recycled ideas.

**When to use:** When the user creates video content and wants automated idea research and tracking.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "ls ${WORKSPACE}/skills/knowledge-base/ 2>/dev/null"
cmd --session <id> --command "ls ${WORKSPACE}/skills/x-research-v2/ 2>/dev/null"
cmd --session <id> --command "echo $ASANA_PAT | head -c 8"
```
- Is the knowledge base deployed? (Used for research queries)
- Is x-research-v2 installed? (Used for X/Twitter research)
- Is Asana connected? (Default destination for idea cards)
- Does the client use Slack? (Default trigger platform)

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Alternative | When to Adapt |
|-----------|---------|-------------|---------------|
| Trigger platform | Slack (@mention + "potential video idea") | Telegram command, Discord, CLI | Client doesn't use Slack |
| Research: X/Twitter | x-research-v2 via bun + X API | Skip X research, use web search instead | Client doesn't have X API access |
| Research: Knowledge base | KB semantic search | Skip if KB not deployed | Knowledge base not installed |
| Card destination | Asana (Video Pipeline project) | Linear, Notion, Trello, local markdown | Client uses a different project management tool |
| Dedup threshold | 40% semantic similarity = auto-skip | Adjust threshold | Client wants stricter/looser dedup |
| Pitch database | SQLite with embeddings | Same (no good alternative) | Rarely needs changing |
| Feedback learning | Learns from accept/reject | Manual curation only | Client preference |

## Prerequisites

- `deploy-knowledge-base.md` completed (research queries)
- `deploy-messaging-setup.md` completed (Slack connection)
- `deploy-asana.md` completed (Asana task creation)
- X/Twitter research skill installed (x-research-v2, requires bun + `X_BEARER_TOKEN`)

## What Gets Installed

### Video Idea Pipeline Skill (`skills/video-idea-pipeline/`)

- **Trigger:** User says "@assistant potential video idea" in Slack
- **Research:** Full thread read -> X/Twitter research -> KB query
- **Output:** Structured Asana card (Video Pipeline project) with idea, research, sources, angles
- **Completion:** Message posted back in Slack thread with Asana link

### Video Pitches Database (`~/clawd/data/video-pitches.db`)

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

Semantic dedup: 40%+ similarity -> auto-skip (prevents recycled ideas)

Learning: Accept/reject feedback improves future research quality and angle suggestions.

### Scripts

| Script | Purpose |
|--------|---------|
| add.js | Add a pitch with embeddings, auto-ID, auto-slug |
| search.js | Semantic + keyword search (70% cosine, 30% keyword) |
| list.js | List recent pitches with filters |
| update.js | Update status |

### Pipeline Flow

1. User mentions "@assistant potential video idea: <topic>" in Slack
2. Agent reads the full Slack thread for context
3. X/Twitter research runs on the topic (trending angles, audience sentiment)
4. Knowledge base queried for related content and past coverage
5. Semantic dedup check against existing pitches (40% threshold)
6. If not a duplicate: create structured Asana card in Video Pipeline project
7. Post completion message back in Slack thread with Asana link
8. If duplicate: notify in Slack with link to the existing similar pitch

## Steps

### 1. Install Video Idea Pipeline Skill

```
cmd --session <id> --command "openclaw skill install video-idea-pipeline --path ~/clawd/skills/video-idea-pipeline/"
```

### 2. Install X-Research-V2 Skill

The X/Twitter research skill requires bun runtime.

```
cmd --session <id> --command "which bun || curl -fsSL https://bun.sh/install | bash"
cmd --session <id> --command "openclaw skill install x-research-v2 --path ~/clawd/skills/x-research-v2/"
```

Verify X/Twitter credentials:

```
cmd --session <id> --command "grep X_BEARER_TOKEN ~/.openclaw/.env"
```

### 3. Configure Asana Integration

Set the Asana personal access token and Video Pipeline project ID.

```
cmd --session <id> --command "openclaw config set asana.pat '<ASANA_PAT>'"
cmd --session <id> --command "openclaw config set asana.video-pipeline-project 1212455754265217"
```

Verify the Asana connection:

```
cmd --session <id> --command "openclaw asana test --project 1212455754265217"
```

### 4. Initialize Video Pitches Database

Create the SQLite database with vector embeddings support.

```
cmd --session <id> --command "cd ~/clawd/tools/video-pitches && node add.js --init"
```

### 5. Configure Slack Trigger

Set up the Slack listener for the video idea trigger phrase.

```
cmd --session <id> --command "openclaw config set slack.triggers.video-idea '@assistant potential video idea'"
```

Verify Slack socket mode is connected:

```
cmd --session <id> --command "openclaw slack status"
```

### 6. Test with a Manual Pitch

Post a test message in Slack or run the pipeline manually:

```
cmd --session <id> --command "cd ~/clawd/tools/video-pitches && node add.js --title 'Test Pitch' --description 'Testing the video idea pipeline' --source 'manual test'"
```

Check the database:

```
cmd --session <id> --command "cd ~/clawd/tools/video-pitches && node list.js --recent 5"
```

Check Asana for the new task:

```
cmd --session <id> --command "openclaw asana list-tasks --project 1212455754265217 --limit 5"
```

## Verification

Run these commands to confirm the video idea pipeline is fully operational:

```
cmd --session <id> --command "cd ~/clawd/tools/video-pitches && node list.js --recent 5"
cmd --session <id> --command "cd ~/clawd/tools/video-pitches && node search.js 'test topic'"
cmd --session <id> --command "openclaw slack status"
```

For an end-to-end test, post "@assistant potential video idea: <topic>" in Slack and verify:
- Agent reacts with eyes emoji
- Research runs (X/Twitter + KB)
- Asana card created in Video Pipeline project
- Completion message posted back in Slack thread with Asana link
- Pitch logged in video-pitches.db

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Slack trigger not firing | Socket mode disconnected or bot not mentioned | Check `openclaw slack status`. Verify the bot is mentioned with the exact trigger phrase. |
| X/Twitter research empty | X_BEARER_TOKEN invalid or expired | Verify token: `grep X_BEARER_TOKEN ~/.openclaw/.env`. Test manually: `cd ~/clawd/skills/x-research-v2 && bun run research.js "test topic"`. |
| KB results empty | Knowledge base not populated | Ingest content into the knowledge base first via `deploy-knowledge-base.md`. |
| Asana card not created | ASANA_PAT invalid or project ID wrong | Verify PAT: `openclaw asana test`. Verify project ID: 1212455754265217. Check Asana permissions. |
| Semantic dedup false positive | Similarity threshold too low | The default is 40%. If legitimate ideas are being flagged as duplicates, increase the threshold in add.js. |
| Embedding errors | GEMINI_API_KEY not set | Verify `GEMINI_API_KEY` in `~/.openclaw/.env`. The pitch database uses gemini-embedding-001 for vectors. |
| All pitches marked "duplicate" | Database has too many broad pitches | Review existing pitches: `node list.js --status pitched`. Consider archiving or rejecting old generic pitches. |

## Dependencies

- **Requires:** `deploy-knowledge-base.md`, `deploy-messaging-setup.md`, `deploy-asana.md`
- **Requires skill:** x-research-v2
