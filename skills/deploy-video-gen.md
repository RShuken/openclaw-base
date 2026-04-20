---
name: "Deploy Video Generation"
category: "integration"
subcategory: "content-gen"
third_party_service: "Gemini API (Veo 3)"
auth_type: "api-key"
openclaw_version_required: "2026.4.15+"
version_last_verified: "2026-04-20"
last_checked: "2026-04-20"
maintenance_status: "active"
model_env_var: "n/a"
clawhub_alternative: "video-generation-minimax"
cost_tier: "api-usage"
privacy_tier: "cloud"
requires:
  - "Gemini API key with Veo 3 access"
  - "HTTP client timeout >= 5 minutes"
docs_urls:
  - "https://ai.google.dev/gemini-api/docs/video"
  - "https://deepmind.google/technologies/veo/"
---

# Deploy Video Generation

## Compatibility
- **OpenClaw Version**: 2026.4.15+ (bumped 2026-04-20)
- **Status**: NEEDS AUDIT
- **Blocked Commands**: (none identified -- no `openclaw` CLI commands referenced)
- **Notes**: This skill verifies API access and writes workspace configuration. No `openclaw` CLI commands found. Should work if the Gemini API key is configured in auth-profiles.json.

## Purpose

Install video generation capabilities via Veo 3. Supports text-to-video and image-to-video for short clips used in social content and visual assets. This gives the client on-demand video creation directly from their agent, powered by the Google AI API.

**When to use:** When the client needs on-demand short video clip generation for social media, thumbnails, presentations, or visual content.

**What this skill does:**
1. Verifies Veo 3 API access via the existing Gemini API key
2. Creates the video generation skill directory and scripts
3. Tests text-to-video generation
4. Tests image-to-video generation (optional)
5. Verifies end-to-end functionality

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

**Important — long-running commands:** Video generation requests are asynchronous and can take 30 seconds to 3 minutes to complete. Do NOT assume the command has timed out or failed if it takes over a minute. If your HTTP client has a timeout, set it to at least 5 minutes for generation commands.

## Variables

Resolve these before executing. Sources are listed for each.

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${GEMINI_API_KEY}` | Client profile or `.env` on client machine | `AIzaSy...` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**

Check if `GEMINI_API_KEY` is configured:

**Remote:**
```
grep GEMINI_API_KEY ${WORKSPACE}/.env | head -c 20
```

Expected: First 20 characters of the key line (confirms key exists without exposing it).

If this fails: The Gemini API key is not set. Configure it in `${WORKSPACE}/.env` before proceeding. The key must have Veo 3 access enabled on the Google AI account.

Check if the video generation skill is already deployed:

**Remote:**
```
test -d ${WORKSPACE}/skills/video-gen && echo 'EXISTS' || echo 'NOT_DEPLOYED'
```

Expected: `NOT_DEPLOYED` for fresh installs, `EXISTS` if previously deployed.

If already exists: Check current files and decide whether to update or skip. Back up existing content before overwriting.

## Adaptation Points

The defaults below work for most installs. Cross-reference against the client profile for overrides.

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Video API | Google Veo 3 (via Gemini API) | Client prefers Runway Gen-3, Pika, or Kling instead |
| Output format | MP4 | Client needs WebM or MOV |
| Output directory | `${WORKSPACE}/output/videos/` | Client wants generated videos stored elsewhere |
| Generation script location | `${WORKSPACE}/skills/video-gen/` | Client has a different skill layout |
| Use case scope | Short clips (5-15s) for social/thumbnails | Client needs longer form content or different aspect ratios |

## Prerequisites

- OpenClaw installed and running (`openclaw/install.md`)
- `GEMINI_API_KEY` configured in `${WORKSPACE}/.env` with Veo 3 access enabled on the Google AI account
- Veo 3 API access approved (may require waitlist or specific API enablement in the Google AI console)

## What Gets Installed

| Component | Location | Purpose |
|-----------|----------|---------|
| Video generation scripts | `${WORKSPACE}/skills/video-gen/` | Generation logic, API interaction, polling |
| Output directory | `${WORKSPACE}/output/videos/` | Default destination for generated clips |

### Capabilities

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

Veo 3 is accessed through the Google AI API using the same `GEMINI_API_KEY` as image generation. Video generation is asynchronous: the request is submitted, and the API returns a generation ID. The skill polls for completion and downloads the result when ready. Expect generation times of 30 seconds to several minutes depending on prompt complexity.

## Steps

### Phase 1: Verify API Access

#### 1.1 Confirm Gemini API Key Has Veo 3 Access `[AUTO]`

Verify the Gemini API key is present and has Veo 3 capabilities. The key is the same one used by `deploy-image-gen.md` if that skill is already deployed.

**Remote:**
```
grep GEMINI_API_KEY ${WORKSPACE}/.env
```

Expected: A line like `GEMINI_API_KEY=AIzaSy...` with a valid key value.

If this fails: The key is missing. Add it to `${WORKSPACE}/.env`. The client needs to create or retrieve their Gemini API key from the Google AI Studio console (https://aistudio.google.com/apikey).

If already exists: Confirm the key has Veo 3 access. If the client already uses `deploy-image-gen.md`, the key is likely present but may not have Veo 3 enabled. Veo 3 access may require separate enablement in the Google AI console.

> **Note:** If Veo 3 is not yet enabled on the account, the operator should guide the client to request access through the Google AI console. Veo 3 may require specific API access approval or a waitlist.

### Phase 2: Install Video Generation Skill

#### 2.1 Create the Skill Directory Structure `[AUTO]`

Create the directory for the video generation skill and an output directory for generated videos.

**Remote:**
```
mkdir -p ${WORKSPACE}/skills/video-gen/scripts ${WORKSPACE}/output/videos
```

Expected: Directories created without error. This command is idempotent (`mkdir -p`).

#### 2.2 Write the Video Generation Script `[GUIDED]`

Write the generation script that handles both text-to-video and image-to-video via the Veo 3 API. This script submits a generation request, polls for completion, and downloads the result.

The script should:
- Accept `--prompt` (required), `--image` (optional, for image-to-video), and `--output` (required) arguments
- Read `GEMINI_API_KEY` from the environment or from `${WORKSPACE}/.env`
- Submit the generation request to the Veo 3 API endpoint
- Poll for completion (check every 10 seconds, timeout after 5 minutes)
- Download the resulting MP4 to the specified output path
- Print progress messages during polling so the operator knows it is still working

**Remote:**
```
cat > ${WORKSPACE}/skills/video-gen/scripts/generate.js << 'EOF'
#!/usr/bin/env node
// Video generation script using Veo 3 via Google AI API
// Usage:
//   node generate.js --prompt "description" --output "/path/to/output.mp4"
//   node generate.js --image "/path/to/image.png" --prompt "motion description" --output "/path/to/output.mp4"

const fs = require('fs');
const path = require('path');

// Parse arguments
const args = process.argv.slice(2);
function getArg(name) {
  const idx = args.indexOf(name);
  return idx !== -1 && idx + 1 < args.length ? args[idx + 1] : null;
}

const prompt = getArg('--prompt');
const imagePath = getArg('--image');
const outputPath = getArg('--output');

if (!prompt || !outputPath) {
  console.error('Usage: node generate.js --prompt "description" --output "path.mp4" [--image "path.png"]');
  process.exit(1);
}

// Load API key from environment or .env file
let apiKey = process.env.GEMINI_API_KEY;
if (!apiKey) {
  try {
    const envFile = fs.readFileSync(path.join(__dirname, '..', '..', '..', '.env'), 'utf8');
    const match = envFile.match(/GEMINI_API_KEY=(.+)/);
    if (match) apiKey = match[1].trim();
  } catch (e) { /* ignore */ }
}

if (!apiKey) {
  console.error('Error: GEMINI_API_KEY not found in environment or .env');
  process.exit(1);
}

async function generate() {
  const model = 'veo-3.0-generate-preview';
  const url = `https://generativelanguage.googleapis.com/v1beta/models/${model}:predictLongRunning?key=${apiKey}`;

  const requestBody = {
    instances: [{
      prompt: prompt
    }],
    parameters: {
      sampleCount: 1
    }
  };

  // If image-to-video, add image data
  if (imagePath) {
    const imageData = fs.readFileSync(imagePath);
    const base64Image = imageData.toString('base64');
    const mimeType = imagePath.endsWith('.png') ? 'image/png' : 'image/jpeg';
    requestBody.instances[0].image = {
      bytesBase64Encoded: base64Image,
      mimeType: mimeType
    };
  }

  console.log(`Submitting ${imagePath ? 'image-to-video' : 'text-to-video'} generation request...`);
  console.log(`Prompt: "${prompt}"`);

  const submitRes = await fetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(requestBody)
  });

  if (!submitRes.ok) {
    const errText = await submitRes.text();
    console.error(`API error (${submitRes.status}): ${errText}`);
    process.exit(1);
  }

  const operation = await submitRes.json();
  const operationName = operation.name;
  console.log(`Generation started: ${operationName}`);
  console.log('Polling for completion (this may take 1-3 minutes)...');

  // Poll for completion
  const pollUrl = `https://generativelanguage.googleapis.com/v1beta/${operationName}?key=${apiKey}`;
  const maxAttempts = 30; // 30 * 10s = 5 minutes
  for (let i = 0; i < maxAttempts; i++) {
    await new Promise(r => setTimeout(r, 10000));
    const pollRes = await fetch(pollUrl);
    const pollData = await pollRes.json();

    if (pollData.done) {
      if (pollData.error) {
        console.error('Generation failed:', JSON.stringify(pollData.error));
        process.exit(1);
      }

      // Extract video data and save
      const videoData = pollData.response?.generatedSamples?.[0]?.video?.bytesBase64Encoded;
      if (!videoData) {
        console.error('No video data in response');
        console.error('Full response:', JSON.stringify(pollData, null, 2));
        process.exit(1);
      }

      const outputDir = path.dirname(outputPath);
      if (!fs.existsSync(outputDir)) fs.mkdirSync(outputDir, { recursive: true });
      fs.writeFileSync(outputPath, Buffer.from(videoData, 'base64'));
      const stats = fs.statSync(outputPath);
      console.log(`Video saved to ${outputPath} (${(stats.size / 1024 / 1024).toFixed(1)} MB)`);
      return;
    }

    const elapsed = (i + 1) * 10;
    console.log(`  Still generating... (${elapsed}s elapsed)`);
  }

  console.error('Generation timed out after 5 minutes');
  process.exit(1);
}

generate().catch(err => {
  console.error('Unexpected error:', err.message);
  process.exit(1);
});
EOF
```

Expected: File written at `${WORKSPACE}/skills/video-gen/scripts/generate.js`.

If this fails: Check that the `${WORKSPACE}/skills/video-gen/scripts/` directory exists. Also check the 8KB command limit -- if the heredoc exceeds it, split the write into chunks or use base64 encoding (see `e2e/lib/test-helpers.ts:writeFileCommands()` for the pattern).

If already exists: Compare with existing content. If unchanged, skip. If different, back up existing as `generate.js.bak` and write the new version.

> **Note on command size:** This heredoc is large. If it exceeds the 8192-byte session API payload limit, the operator should use chunked file writing or base64 encoding to deliver the file content.

#### 2.3 Make the Script Executable `[AUTO]`

**Remote (macOS/Linux):**
```
chmod +x ${WORKSPACE}/skills/video-gen/scripts/generate.js
```

Expected: No output (success). Not needed on Windows.

### Phase 3: Test Video Generation

> **Important — async timing:** Video generation commands take 30 seconds to 3 minutes to complete. The session API will hold the connection open until the command finishes. Do NOT assume timeout or failure if the command takes over a minute. The progress messages from the script will appear in the command output when it finally returns.

#### 3.1 Test Text-to-Video `[GUIDED]`

Generate a simple test clip to confirm the full pipeline works end-to-end. This step will take 1-3 minutes.

**Remote:**
```
cd ${WORKSPACE}/skills/video-gen && node scripts/generate.js --prompt 'a lobster walking on a beach at sunset' --output '${WORKSPACE}/output/videos/test-video.mp4'
```

Expected: Script prints progress updates during polling, then saves an MP4 file. Final output line shows the file path and size (typically 1-10 MB for a short clip).

If this fails:
- **403 error:** Veo 3 access is not enabled on the API key's account. Check the Google AI console and request Veo 3 access.
- **API key invalid:** Verify the key in `${WORKSPACE}/.env` is correct.
- **Timeout (5+ minutes):** The API may be overloaded. Retry with a simpler prompt (e.g., "a spinning red cube on white background").
- **Rate limit / quota exceeded:** Wait a few minutes and retry. Check API usage limits in the Google AI console.
- **Network error:** Verify the client machine has internet access and can reach `generativelanguage.googleapis.com`.

#### 3.2 Verify Test Output `[AUTO]`

Confirm the test video was created and has valid content.

**Remote:**
```
ls -la ${WORKSPACE}/output/videos/test-video.mp4
```

Expected: File exists with a reasonable size (typically 1-10 MB). A 0-byte file indicates the download or generation failed.

If this fails: The generation did not produce output. Check the script's error messages from the previous step. Common causes: API returned an error, response format changed, or the video data field was empty.

#### 3.3 Test Image-to-Video (Optional) `[GUIDED]`

If the client needs image-to-video capabilities, test animating a still image. This requires an existing image file on the client machine.

**Remote:**
```
cd ${WORKSPACE}/skills/video-gen && node scripts/generate.js --image '/tmp/test-image.png' --prompt 'slowly zoom in and add gentle motion' --output '${WORKSPACE}/output/videos/test-animate.mp4'
```

Expected: Same polling behavior as text-to-video. Output MP4 shows the input image with the described motion applied.

If this fails:
- **Image not found:** Ensure the image path is correct and the file exists on the client machine.
- **Unsupported format:** Input must be PNG or JPEG. Convert other formats first.
- **Image too large:** Resize to under 4096x4096 if needed.

#### 3.4 Clean Up Test Files `[AUTO]`

Remove test output files after verification.

**Remote:**
```
rm -f ${WORKSPACE}/output/videos/test-video.mp4 ${WORKSPACE}/output/videos/test-animate.mp4
```

Expected: Test files removed. The output directory remains for future use.

### Phase 4: Post-Deployment

#### 4.1 Update Client Profile `[AUTO]`

Update `clients/<name>.md` with:
- Video generation deployed (Veo 3)
- Skill location: `${WORKSPACE}/skills/video-gen/`
- Output directory: `${WORKSPACE}/output/videos/`
- Any issues encountered during setup

#### 4.2 Note Follow-Ups `[AUTO]`

Inform the operator of related capabilities:
- `deploy-image-gen.md` pairs well -- generate still images, then animate them with video gen
- `deploy-video-analysis.md` can analyze generated videos for quality checks
- `deploy-video-pipeline.md` can integrate video generation into automated content workflows

## Verification

Run a complete generation cycle to confirm the system works:

**Remote:**
```
cd ${WORKSPACE}/skills/video-gen && node scripts/generate.js --prompt "a simple spinning cube on a white background" --output "${WORKSPACE}/output/videos/verify-video.mp4"
```

Expected:
- Command completes within 1-3 minutes (do NOT assume timeout)
- Progress messages appear during polling
- Output file exists and is a valid MP4 (1-10 MB)

**Remote:**
```
ls -la ${WORKSPACE}/output/videos/verify-video.mp4
```

Expected: File exists with non-zero size.

Clean up after verification:

**Remote:**
```
rm -f ${WORKSPACE}/output/videos/verify-video.mp4
```

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| API not available / 403 error | Veo 3 access not enabled on the account | Check the Google AI console. Veo 3 may require specific access approval or waitlist. |
| GEMINI_API_KEY not found | Key missing from environment and `.env` | Add `GEMINI_API_KEY=...` to `${WORKSPACE}/.env` |
| Generation very slow (5+ minutes) | Normal for complex prompts; API is processing | Wait longer. If it exceeds 10 minutes, the generation may have failed silently. Retry with a simpler prompt. |
| Quality issues (artifacts, blurry) | Prompt too vague or overly complex | Use shorter, more specific prompts. Describe a single clear scene rather than multiple actions. |
| File size too large | High resolution or long duration | Reduce resolution or shorten the clip duration in the request parameters. |
| Image-to-video fails | Input image format not supported or too large | Ensure the input image is PNG or JPEG. Resize to under 4096x4096 if needed. |
| Rate limiting / quota exceeded | Too many requests in a short period | Wait and retry. Check API usage limits in the Google AI console. |
| Script heredoc exceeds 8KB limit | File content too large for single API command | Use chunked writes or base64 encoding (see `e2e/lib/test-helpers.ts:writeFileCommands()`). |
| Command appears to hang (1-3 min) | Normal — video generation is slow | Do NOT cancel. Wait for the full response. Generation typically takes 30s-3min. |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Verify API key | Execute silently | Execute silently | Show key check, confirm |
| 2.1 Create directories | Execute silently | Execute silently | Confirm before creating |
| 2.2 Write generation script | Execute silently | Show script content, confirm before writing | Show script content, confirm |
| 2.3 Make executable | Execute silently | Execute silently | Confirm |
| 3.1 Test text-to-video | Execute silently (expect 1-3 min wait) | Confirm before generating | Confirm, explain expected wait time |
| 3.2 Verify output | Execute silently | Execute silently | Show file details, confirm |
| 3.3 Image-to-video test | Skip unless client needs it | Ask if client wants this tested | Ask if client wants this tested |
| 3.4 Clean up test files | Execute silently | Execute silently | Confirm before deleting |
| 4.1 Update client profile | Execute silently | Execute silently | Show changes, confirm |

## Dependencies

- **Depends on:** OpenClaw installed (`openclaw/install.md`), `GEMINI_API_KEY` with Veo 3 access
- **Independent of:** All other deploy skills (can be deployed standalone)
- **Pairs well with:** `deploy-image-gen.md` (generate still images, then animate them with video gen), `deploy-video-analysis.md` (analyze generated videos), `deploy-video-pipeline.md` (automated content workflows)
