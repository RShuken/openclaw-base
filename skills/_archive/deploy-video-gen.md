# Deploy Video Generation

## Purpose

Install video generation capabilities via Veo 3. Supports text-to-video and image-to-video for short clips used in social content and visual assets.

**When to use:** When the user needs on-demand short video clip generation.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "echo $GEMINI_API_KEY | head -c 8"
```
- Is GEMINI_API_KEY configured with Veo 3 access?
- Does the client need video generation?

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Alternative | When to Adapt |
|-----------|---------|-------------|---------------|
| Video API | Google Veo 3 (via Gemini API) | Runway Gen-3, Pika, Kling | Client prefers a different video provider or Veo 3 isn't available |
| Output format | MP4 | WebM, MOV | Client needs a specific format |
| Use case | Short clips for social/thumbnails | Longer form content | Client has different video needs |

## Prerequisites

- `GEMINI_API_KEY` configured in `.env` (Veo 3 is accessed via Google AI APIs)
- Veo 3 API access enabled on the Google AI account

## What Gets Installed

### Video Generation Integration

| Capability | Description |
|------------|-------------|
| Text-to-video | Generate short video clips from text descriptions |
| Image-to-video | Animate a still image into a video clip |
| Resolution control | Configurable output resolution |
| Duration control | Short clips (5-15 seconds typical) |

### Use Cases

- YouTube thumbnails animated into short intros
- Social media video clips for Instagram Reels, TikTok, X/Twitter
- Visual assets for presentations and newsletters
- Product demos and explainer snippets

### How It Works

Veo 3 is accessed through the Google AI API using the same `GEMINI_API_KEY` as image generation. Video generation is asynchronous: the request is submitted, and the API returns a generation ID. The skill polls for completion and downloads the result when ready. Expect generation times of 30 seconds to several minutes depending on complexity.

## Steps

### 1. Verify Veo 3 API Access

Check that the `GEMINI_API_KEY` has Veo 3 access enabled.

```
cmd --session <id> --command "grep GEMINI_API_KEY ~/clawd/.env"
```

If Veo 3 is not enabled on the account, request access through the Google AI console. Veo 3 may require specific API access or waitlist approval.

### 2. Install Video Generation Skill

```
cmd --session <id> --command "cp -r /path/to/video-gen ~/clawd/skills/video-gen"
```

Verify the structure:

```
cmd --session <id> --command "ls ~/clawd/skills/video-gen/"
```

### 3. Test Text-to-Video

Generate a simple test clip:

```
cmd --session <id> --command "cd ~/clawd/skills/video-gen && node scripts/generate.js --prompt 'a lobster walking on a beach at sunset' --output '/tmp/test-video.mp4'"
```

This may take 1-3 minutes. The script will poll the API and download the result when ready.

### 4. Verify Output

```
cmd --session <id> --command "ls -la /tmp/test-video.mp4"
```

The file should exist and be a valid MP4. Play it to confirm it matches the prompt.

### 5. Test Image-to-Video (Optional)

If an image is available, test animating it:

```
cmd --session <id> --command "cd ~/clawd/skills/video-gen && node scripts/generate.js --image '/tmp/test-image.png' --prompt 'slowly zoom in and add gentle motion' --output '/tmp/test-animate.mp4'"
```

## Verification

Run a test generation and verify the output:

```
cd ~/clawd/skills/video-gen
node scripts/generate.js --prompt "a simple spinning cube on a white background" --output "/tmp/test-verify-video.mp4"
ls -la /tmp/test-verify-video.mp4
```

Expected:
- Command completes (may take 1-3 minutes)
- Output file exists and is a valid MP4
- Video plays and matches the prompt description

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| API not available / 403 error | Veo 3 access not enabled on the account | Check the Google AI console. Veo 3 may require specific access approval or waitlist. |
| Generation very slow (5+ minutes) | Normal for complex prompts; API is processing | Wait longer. If it exceeds 10 minutes, the generation may have failed silently. Re-try with a simpler prompt. |
| Quality issues (artifacts, blurry output) | Prompt too vague or overly complex | Use shorter, more specific prompts. Describe a single clear scene rather than multiple actions. |
| File size too large | High resolution or long duration | Reduce resolution or shorten the clip duration in the request parameters. |
| Image-to-video fails | Input image format not supported or too large | Ensure the input image is PNG or JPEG. Resize to under 4096x4096 if needed. |
| Rate limiting / quota exceeded | Too many requests in a short period | Wait and retry. Check API usage limits in the Google AI console. |

## Dependencies

- **Requires:** `GEMINI_API_KEY` with Veo 3 access in `.env`
- **Independent of:** All other deploy skills
- **Pairs well with:** `deploy-image-gen.md` (generate still images, then animate them with video gen)
