# Deploy Video Analysis

## Purpose

Install video analysis capabilities via Gemini. Upload any video for AI-powered analysis -- content review, insight extraction, key moment identification, and summarization.

**When to use:** When the user wants to analyze video content (own videos before publishing, competitor analysis, extracting talking points).

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "ls ${WORKSPACE}/skills/gemini-video-watch/ 2>/dev/null"
cmd --session <id> --command "echo $GEMINI_API_KEY | head -c 8"
```
- Is the Gemini Video Watch skill already installed?
- Is GEMINI_API_KEY configured?

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Alternative | When to Adapt |
|-----------|---------|-------------|---------------|
| Analysis API | Google Gemini 3 Pro | OpenAI GPT-4o (vision), Anthropic Claude (vision) | Client prefers a different provider or Gemini not available |
| Input handling | Local files, YouTube URLs, Telegram uploads, direct URLs | Subset based on client's workflows | Client only uses certain input types |
| YouTube support | Native (Gemini fetches directly) | Download first, then upload | Alternative provider may not support native YouTube |
| Temp file cleanup | Manual or `--delete-file` flag | Auto-cleanup always | Client preference |

## Prerequisites

- `GEMINI_API_KEY` configured in `.env`

## What Gets Installed

### Gemini Video Watch Skill (`skills/gemini-video-watch/`)

| Setting | Value |
|---------|-------|
| Entry point | `node scripts/watch.js` |
| API | Gemini Files API (resumable upload) + Gemini model |
| Supported inputs | Local files, YouTube URLs, direct video URLs, Telegram uploads |

### Command

```
node scripts/watch.js "<video_path_or_url>" --prompt "<prompt>" [--model X] [--temperature X] [--delete-file]
```

### Supported Input Types

| Input | How |
|-------|-----|
| Local files | Direct upload from filesystem |
| YouTube URLs | Native support (Gemini fetches directly, no local download needed) |
| Direct video URLs | Auto-download then upload |
| Telegram uploads | Download from Telegram, then upload |

### Features

- **Resumable upload:** Uses the Gemini Files API for large file uploads that can recover from interruptions
- **Native YouTube support:** Pass a YouTube URL directly and Gemini fetches the video itself, no local download step
- **Auto-download:** Direct video URLs are automatically downloaded and uploaded
- **Customizable model:** Override the default model with `--model`
- **Temperature control:** Adjust creativity/precision with `--temperature`
- **Cleanup:** `--delete-file` flag removes the uploaded file from Gemini after analysis

### Use Cases

- Pre-publish review of own videos (catch mistakes, check pacing, verify claims)
- Competitor content analysis (what are they doing, what's working)
- Extracting talking points from interviews and presentations
- Content summarization (turn a 30-minute video into key bullet points)
- Identifying key moments and timestamps

## Steps

### 1. Verify GEMINI_API_KEY

```
cmd --session <id> --command "grep GEMINI_API_KEY ~/clawd/.env"
```

### 2. Install the Gemini Video Watch Skill

```
cmd --session <id> --command "cp -r /path/to/gemini-video-watch ~/clawd/skills/gemini-video-watch"
```

Verify the structure:

```
cmd --session <id> --command "ls ~/clawd/skills/gemini-video-watch/scripts/"
```

Expected: `watch.js` and supporting files.

### 3. Install Dependencies

```
cmd --session <id> --command "cd ~/clawd/skills/gemini-video-watch && npm install"
```

### 4. Test with a YouTube URL

```
cmd --session <id> --command "cd ~/clawd/skills/gemini-video-watch && node scripts/watch.js 'https://youtube.com/watch?v=dQw4w9WgXcQ' --prompt 'Summarize this video in 3 bullet points'"
```

Expected: The skill fetches the video via Gemini's native YouTube support and returns a summary.

### 5. Test with a Local File

```
cmd --session <id> --command "cd ~/clawd/skills/gemini-video-watch && node scripts/watch.js '/path/to/local-video.mp4' --prompt 'What are the key points discussed?'"
```

Expected: The skill uploads the file using resumable upload and returns analysis.

### 6. Test Cleanup Flag

```
cmd --session <id> --command "cd ~/clawd/skills/gemini-video-watch && node scripts/watch.js '/path/to/video.mp4' --prompt 'Brief summary' --delete-file"
```

Expected: Analysis is returned and the uploaded file is deleted from Gemini's storage after.

## Verification

Run a quick test against a known YouTube video:

```
cd ~/clawd/skills/gemini-video-watch
node scripts/watch.js "https://youtube.com/watch?v=dQw4w9WgXcQ" --prompt "What is this video about?"
```

Expected:
- Command completes without errors
- Returns a coherent description of the video content
- No upload timeout errors

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Upload timeout | Large file and slow connection | Resumable upload should recover automatically. If it keeps failing, try a smaller file first to confirm the API works. |
| YouTube URL fails | Video is private, restricted, or region-locked | Try a different public video. Gemini cannot access private or age-restricted content. |
| Gemini quota exceeded | Too many requests or too much data uploaded | Check API usage limits in the Google AI console. Wait for quota reset or request a limit increase. |
| File too large | Exceeds Gemini Files API size limits | Try shorter clips. For long videos, trim to the relevant section before uploading. |
| Analysis quality poor | Prompt too vague | Be specific about what you want from the analysis. "What are the 3 main arguments?" works better than "Analyze this." |
| `--delete-file` didn't clean up | API error during deletion | Check manually in the Google AI console for orphaned files. Not critical but wastes storage. |
| Node.js errors on startup | Dependencies not installed | Run `npm install` in the skill directory. |

## Dependencies

- **Requires:** `GEMINI_API_KEY` in `.env`
- **Independent of:** All other deploy skills
- **Pairs well with:** `deploy-video-gen.md` (analyze existing content, then generate new content based on insights)
