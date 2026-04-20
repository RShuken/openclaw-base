---
name: "Deploy Video Analysis"
category: "integration"
subcategory: "content-gen"
third_party_service: "Gemini API"
auth_type: "api-key"
openclaw_version_required: "2026.4.15+"
version_last_verified: "2026-04-20"
last_checked: "2026-04-20"
maintenance_status: "active"
model_env_var: "VIDEO_ANALYSIS_MODEL"
clawhub_alternative: "none"
cost_tier: "api-usage"
privacy_tier: "cloud"
requires:
  - "Gemini API key"
  - "Gemini Video Watch skill installed"
docs_urls:
  - "https://ai.google.dev/gemini-api/docs/vision"
  - "https://ai.google.dev/gemini-api/docs/video-understanding"
---

# Deploy Video Analysis

## Compatibility
- **OpenClaw Version**: 2026.4.15+ (bumped 2026-04-20)
- **Status**: NEEDS AUDIT
- **Blocked Commands**: (none identified -- no `openclaw` CLI commands referenced)
- **Notes**: This skill primarily verifies API keys and configures workspace files. No `openclaw` CLI commands found. Should work if the Gemini API key is configured in auth-profiles.json. Verify that OpenClaw's video analysis capability is available via `openclaw skills list`.
- **Model**: Gemini is pinned for the video-analysis capability itself (non-Gemini LLMs can't consume video). Override this skill's *text-layer* LLM routing (summaries, insight extraction over the Gemini output) via env var **`VIDEO_ANALYSIS_MODEL`** (see `_authoring/_deploy-common.md` "Per-skill env-var overrides" for the pattern). Leave unset to defer the text layer to the agent default.

## Purpose

Install video analysis capabilities via Gemini. Upload any video for AI-powered analysis -- content review, insight extraction, key moment identification, and summarization. Supports local files, YouTube URLs, direct video URLs, and Telegram uploads.

**When to use:** When the client wants to analyze video content (own videos before publishing, competitor analysis, extracting talking points from interviews and presentations).

**What this skill does:**
1. Verifies the Gemini API key is configured
2. Installs the Gemini Video Watch skill and its dependencies
3. Tests analysis against YouTube URLs, local files, and cleanup behavior
4. Verifies the full pipeline end-to-end

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${GEMINI_API_KEY}` | Client profile or `.env` file on client machine | `AIzaSy...` |
| `${TEST_VIDEO_PATH}` | Any video file on the client's filesystem (for local file test) | `${WORKSPACE}/media/sample.mp4` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**

Check if the Gemini Video Watch skill is already installed:

**Remote:**
```
ls ${WORKSPACE}/skills/gemini-video-watch/ 2>/dev/null && echo 'INSTALLED' || echo 'NOT_INSTALLED'
```

Expected: `NOT_INSTALLED` for a fresh deploy, or `INSTALLED` if already present.

Check if `GEMINI_API_KEY` is configured:

**Remote:**
```
grep GEMINI_API_KEY ${WORKSPACE}/.env 2>/dev/null | head -c 20
```

Expected: First 20 characters of the key line. If empty, the key must be added before proceeding.

**Decision points from pre-flight:**
- Is the skill already installed? If so, check if it needs updating or skip to verification.
- Is `GEMINI_API_KEY` present? If not, obtain it from the client or the client profile and add it to `.env`.

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Analysis API | Google Gemini 3 Pro | Client prefers OpenAI GPT-4o (vision) or Anthropic Claude (vision), or Gemini is not available |
| Input handling | Local files, YouTube URLs, Telegram uploads, direct URLs | Client only uses certain input types -- skip irrelevant tests |
| YouTube support | Native (Gemini fetches directly) | Alternative provider may not support native YouTube; download first, then upload |
| Temp file cleanup | Manual or `--delete-file` flag | Client prefers auto-cleanup always |
| Skill directory | `${WORKSPACE}/skills/gemini-video-watch/` | Client organizes skills differently |

## Prerequisites

- OpenClaw installed and running (`openclaw/install.md`)
- `GEMINI_API_KEY` configured in `${WORKSPACE}/.env`
- Node.js available on the client machine (installed with OpenClaw)

## What Gets Installed

| Component | Location | Purpose |
|-----------|----------|---------|
| Gemini Video Watch skill | `${WORKSPACE}/skills/gemini-video-watch/` | Main skill directory |
| Entry point script | `${WORKSPACE}/skills/gemini-video-watch/scripts/watch.js` | Video analysis CLI |
| Node dependencies | `${WORKSPACE}/skills/gemini-video-watch/node_modules/` | Gemini SDK and utilities |

### Command Interface

```
node scripts/watch.js "<video_path_or_url>" --prompt "<prompt>" [--model X] [--temperature X] [--delete-file]
```

### Supported Input Types

| Input | How |
|-------|-----|
| Local files | Direct upload from filesystem via Gemini Files API (resumable) |
| YouTube URLs | Native support -- Gemini fetches directly, no local download needed |
| Direct video URLs | Auto-download then upload |
| Telegram uploads | Download from Telegram, then upload |

### Features

- **Resumable upload:** Uses the Gemini Files API for large file uploads that can recover from interruptions
- **Native YouTube support:** Pass a YouTube URL directly and Gemini fetches the video itself
- **Auto-download:** Direct video URLs are automatically downloaded and uploaded
- **Customizable model:** Override the default model with `--model`
- **Temperature control:** Adjust creativity/precision with `--temperature`
- **Cleanup:** `--delete-file` flag removes the uploaded file from Gemini after analysis

## Steps

### Phase 1: Configuration

#### 1.1 Verify GEMINI_API_KEY `[AUTO]`

Confirm the Gemini API key is present and valid in the client's environment file.

**Remote:**
```
grep GEMINI_API_KEY ${WORKSPACE}/.env
```

Expected: A line containing `GEMINI_API_KEY=AIzaSy...` (the key value).

If this fails: The key is missing. Obtain it from the client profile or ask the client. Add it to the `.env` file:

**Remote:**
```
echo 'GEMINI_API_KEY=<actual_key>' >> ${WORKSPACE}/.env
```

If already exists: Confirm the key is not empty or placeholder text. If it looks valid, skip.

### Phase 2: Install Skill

#### 2.1 Create the Skill Directory `[AUTO]`

Create the directory structure for the Gemini Video Watch skill.

**Remote:**
```
mkdir -p ${WORKSPACE}/skills/gemini-video-watch/scripts
```

Expected: Directory exists at `${WORKSPACE}/skills/gemini-video-watch/scripts/`.

If already exists: Safe to skip -- `mkdir -p` is idempotent.

#### 2.2 Deploy Skill Files `[GUIDED]`

Copy the Gemini Video Watch skill files into the skill directory. The source location depends on how the skill is distributed -- it may be in a template directory, a git repo, or provided by the operator.

Write the entry point script (`watch.js`) and any supporting files to `${WORKSPACE}/skills/gemini-video-watch/scripts/`.

**Remote:**
```
ls ${WORKSPACE}/skills/gemini-video-watch/scripts/
```

Expected: `watch.js` and any supporting files are present.

If this fails: The skill files were not copied correctly. Verify the source path exists and re-copy. Check file permissions if the copy command succeeded but files are not readable.

If already exists: Compare file contents or version markers. If unchanged, skip. If different, back up existing as `.bak` and write new version.

#### 2.3 Install Dependencies `[AUTO]`

Install the Node.js dependencies required by the skill (Gemini SDK and utilities).

**Remote:**
```
cd ${WORKSPACE}/skills/gemini-video-watch && npm install
```

Expected: `npm install` completes without errors and `node_modules/` directory is populated.

If this fails: Check that `package.json` exists in the skill directory. Check that Node.js and npm are available (`node --version && npm --version`). If behind a corporate proxy, npm may need proxy configuration.

If already exists: Running `npm install` again is safe and idempotent -- it resolves any missing or outdated packages.

### Phase 3: Test and Verify

#### 3.1 Test with a YouTube URL `[GUIDED]`

Verify the skill works with Gemini's native YouTube support. Use any public YouTube video -- the operator should pick a short, well-known public video to keep the test quick.

**Remote:**
```
cd ${WORKSPACE}/skills/gemini-video-watch && node scripts/watch.js 'https://youtube.com/watch?v=dQw4w9WgXcQ' --prompt 'Summarize this video in 3 bullet points'
```

Expected: The skill fetches the video via Gemini's native YouTube support and returns a coherent multi-point summary. No upload timeout errors.

If this fails:
- **API key error:** Verify `GEMINI_API_KEY` is correct and not expired. Test the key directly with a simple Gemini API call.
- **YouTube URL fails:** The video may be private, restricted, or region-locked. Try a different public video.
- **Quota exceeded:** Check API usage limits in the Google AI console. Wait for quota reset or request increase.
- **Timeout:** Network issues. Retry once; if persistent, check the client's internet connectivity.

#### 3.2 Test with a Local File `[GUIDED]`

Verify the skill works with local file upload via the Gemini Files API. The operator should identify a video file that already exists on the client's machine, or use a small test file.

First, find a video file on the client's machine to use for testing:

**Remote:**
```
find ${WORKSPACE} -name '*.mp4' -o -name '*.mov' -o -name '*.webm' 2>/dev/null | head -5
```

If no video files exist, download a small public-domain test clip or skip this test if the YouTube test passed and local file analysis is not a priority for this client.

Then run the analysis against the identified file:

**Remote:**
```
cd ${WORKSPACE}/skills/gemini-video-watch && node scripts/watch.js '${TEST_VIDEO_PATH}' --prompt 'What are the key points discussed?'
```

Expected: The skill uploads the file using the resumable upload API and returns a coherent analysis of the video content.

If this fails:
- **File not found:** Double-check the path. Use an absolute path, not relative.
- **File too large:** Gemini Files API has size limits. Try a shorter clip or trim the video first.
- **Upload timeout:** Large file on a slow connection. The resumable upload should recover automatically. If it keeps failing, try a smaller file to confirm the API works.

#### 3.3 Test Cleanup Flag `[AUTO]`

Verify that the `--delete-file` flag properly removes the uploaded file from Gemini's storage after analysis. Use the same local file from the previous test, or a YouTube URL.

**Remote:**
```
cd ${WORKSPACE}/skills/gemini-video-watch && node scripts/watch.js '${TEST_VIDEO_PATH}' --prompt 'Brief summary' --delete-file
```

Expected: Analysis is returned and the uploaded file is deleted from Gemini's storage afterward. No orphaned files left in Gemini.

If this fails: The analysis itself should still complete. If the deletion step fails, it is not critical -- check the Google AI console for orphaned files manually. This is a storage hygiene issue, not a functional blocker.

## Verification

Run these checks to confirm the full pipeline is working:

**Remote:**
```
cd ${WORKSPACE}/skills/gemini-video-watch && node scripts/watch.js 'https://youtube.com/watch?v=dQw4w9WgXcQ' --prompt 'What is this video about?'
```

Expected:
- Command completes without errors
- Returns a coherent description of the video content
- No upload timeout errors

**Remote:**
```
ls ${WORKSPACE}/skills/gemini-video-watch/scripts/watch.js && ls ${WORKSPACE}/skills/gemini-video-watch/node_modules/
```

Expected: Both the entry point script and `node_modules/` directory exist.

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Upload timeout | Large file and slow connection | Resumable upload should recover automatically. If it keeps failing, try a smaller file first to confirm the API works. |
| YouTube URL fails | Video is private, restricted, or region-locked | Try a different public video. Gemini cannot access private or age-restricted content. |
| Gemini quota exceeded | Too many requests or too much data uploaded | Check API usage limits in the Google AI console. Wait for quota reset or request a limit increase. |
| File too large | Exceeds Gemini Files API size limits | Try shorter clips. For long videos, trim to the relevant section before uploading. |
| Analysis quality poor | Prompt too vague | Be specific about what you want. "What are the 3 main arguments?" works better than "Analyze this." |
| `--delete-file` didn't clean up | API error during deletion | Check manually in the Google AI console for orphaned files. Not critical but wastes storage. |
| Node.js errors on startup | Dependencies not installed | Run `npm install` in the skill directory. |
| `GEMINI_API_KEY` not found | Key missing from `.env` | Add it: `echo 'GEMINI_API_KEY=<key>' >> ${WORKSPACE}/.env` |
| Permission denied on script | File permissions wrong | `chmod +x ${WORKSPACE}/skills/gemini-video-watch/scripts/watch.js` |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Verify API Key | Execute silently | Confirm key is present | Show key (masked), confirm |
| 2.1 Create Skill Directory | Execute silently | Execute silently | Confirm before creating |
| 2.2 Deploy Skill Files | Execute silently | Confirm before deploying | Confirm, show file list |
| 2.3 Install Dependencies | Execute silently | Execute silently | Confirm before npm install |
| 3.1 Test YouTube URL | Execute silently | Execute, show results | Confirm before testing, show results |
| 3.2 Test Local File | Execute silently | Ask which file to use | Ask which file, confirm command |
| 3.3 Test Cleanup Flag | Execute silently | Execute, show results | Confirm before testing |

## Dependencies

- **Depends on:** `openclaw/install.md` (OpenClaw must be installed), `GEMINI_API_KEY` in `.env`
- **Required by:** None -- this is a standalone analysis capability
- **Pairs well with:** `deploy-video-gen.md` (analyze existing content, then generate new content based on insights), `deploy-knowledge-base.md` (ingest video insights into the knowledge base)
