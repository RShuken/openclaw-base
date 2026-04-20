---
name: "Deploy Image Generation"
category: "integration"
subcategory: "content-gen"
third_party_service: "Gemini API (Nano Banana Pro)"
auth_type: "api-key"
openclaw_version_required: "2026.4.15+"
version_last_verified: "2026-04-20"
last_checked: "2026-04-20"
maintenance_status: "active"
model_env_var: "n/a"
clawhub_alternative: "nano-banana-pro"
cost_tier: "api-usage"
privacy_tier: "cloud"
requires:
  - "Gemini API key"
  - "uv Python package manager"
docs_urls:
  - "https://ai.google.dev/gemini-api/docs/image-generation"
  - "https://ai.google.dev/gemini-api/docs"
---

# Deploy Image Generation

## Compatibility
- **OpenClaw Version**: 2026.4.15+ (bumped 2026-04-20)
- **Status**: NEEDS AUDIT
- **Blocked Commands**: (none identified -- no `openclaw` CLI commands referenced)
- **Notes**: This skill primarily verifies API keys and writes workspace files. No `openclaw` CLI commands found in the skill steps. Should work if the Gemini API key is configured in auth-profiles.json. Verify that OpenClaw's image generation skill is available via `openclaw skills list`.

## Purpose

Install image generation capabilities via Google Gemini 3 Pro Image API (Nano Banana). Supports text-to-image, image editing, and multi-image composition for thumbnails, social posts, and visual assets.

**When to use:** When a client needs on-demand image generation from text prompts, image editing, or multi-image composition.

**What this skill does:**
1. Verifies the Gemini API key is configured
2. Installs the `uv` Python package manager if missing
3. Deploys the Nano Banana Pro 2 skill (generate_image.py + pyproject.toml)
4. Tests text-to-image, editing, and composition capabilities

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${GEMINI_API_KEY}` | Client's Gemini API key (from client profile or `[HUMAN_INPUT]`) | `AIzaSy...` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**

Check if `uv` is already installed:

**Remote (macOS/Linux):**
```
which uv 2>/dev/null && uv --version || echo 'NOT_INSTALLED'
```

**Remote (Windows):**
```
try { uv --version } catch { 'NOT_INSTALLED' }
```

Check if the Gemini API key is configured:

**Remote (macOS/Linux):**
```
grep GEMINI_API_KEY ${WORKSPACE}/.env 2>/dev/null | head -c 20 || echo 'NOT_SET'
```

**Remote (Windows):**
```
if (Test-Path "${WORKSPACE}\.env") { Select-String -Path "${WORKSPACE}\.env" -Pattern 'GEMINI_API_KEY' | ForEach-Object { $_.Line.Substring(0,20) } } else { 'NOT_SET' }
```

Check if the skill directory already exists:

**Remote (macOS/Linux):**
```
ls ${WORKSPACE}/skills/nano-banana-pro-2/scripts/generate_image.py 2>/dev/null && echo 'EXISTS' || echo 'NOT_FOUND'
```

**Decision points from pre-flight:**
- Is `uv` already installed? If yes, skip install.
- Is `GEMINI_API_KEY` configured? If not, obtain it from the client.
- Is the skill directory already deployed? If yes, check if it needs updating.
- Does the client actually need image generation?

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Image API | Google Gemini 3 Pro (Nano Banana) | Client prefers OpenAI DALL-E 3, Stability AI, or Midjourney API |
| Python runtime | `uv` (auto-manages dependencies) | Client uses pip or conda instead |
| Default resolution | 1K | Client typically needs 2K or 4K output |
| Output path | Timestamped files in working directory | Client wants a dedicated asset output directory |
| Skill location | `${WORKSPACE}/skills/nano-banana-pro-2/` | Client organizes skills differently |

## Prerequisites

- OpenClaw installed and running (`openclaw/install.md`)
- `GEMINI_API_KEY` configured in `${WORKSPACE}/.env` (obtain from Google AI Studio if missing)
- Python 3 available on the client machine
- `uv` Python package manager (installed in Phase 1 if missing)

## What Gets Installed

### Nano Banana Pro 2 Skill (`${WORKSPACE}/skills/nano-banana-pro-2/`)

| Command | Purpose |
|---------|---------|
| `uv run scripts/generate_image.py --prompt "<desc>" --filename "out.png"` | Text-to-image |
| Same with `-i <input.png>` | Image editing (modify existing image) |
| Multiple `-i` flags | Multi-image composition (up to 14 images) |
| `--resolution 1K\|2K\|4K` | Output resolution control |

### Features

- **Text-to-image generation:** Describe what you want, get an image
- **Single image editing:** Pass an existing image with `-i` plus a prompt to modify it
- **Multi-image composition:** Combine up to 14 input images into a new composition
- **Resolution control:** 1K (default), 2K, or 4K output
- **Timestamped filenames:** Auto-generates unique filenames when `--filename` is omitted
- **Agent runtime integration:** Outputs `MEDIA:<path>` format for agent runtimes that support inline media

### How It Works

The skill wraps the Google Gemini 3 Pro Image API. It sends the prompt (and optional input images) to the API, receives the generated image, and saves it to disk. The `uv` package manager handles all Python dependencies automatically via `pyproject.toml`, so there is no manual `pip install` step.

## Steps

### Phase 1: Environment Setup

#### 1.1 Verify Gemini API Key `[HUMAN_INPUT]`

Check that the Gemini API key is present in the workspace `.env` file. This key is required for all image generation calls.

**Remote:**
```
grep GEMINI_API_KEY ${WORKSPACE}/.env
```

Expected: A line like `GEMINI_API_KEY=AIzaSy...` with a valid key value.

If this fails: The key is missing. Obtain it from the client or from Google AI Studio (https://aistudio.google.com/apikey). The key must have Imagen/image generation access enabled.

**Remote (to add the key):**
```
printf '\nGEMINI_API_KEY=%s\n' '${GEMINI_API_KEY}' >> ${WORKSPACE}/.env
```

> **Important:** Substitute the actual API key value before sending. The session API does not expand variables.

If already exists: Verify the key is valid (non-empty, starts with expected prefix). If correct, skip.

#### 1.2 Install uv (Python Package Installer) `[AUTO]`

Install `uv`, the fast Python package manager that handles all dependencies for the image generation script. `uv` resolves dependencies from `pyproject.toml` automatically on first run, so no manual `pip install` is needed.

**Remote (macOS — Homebrew):**
```
which uv || brew install uv
```

**Remote (Linux):**
```
which uv || curl -LsSf https://astral.sh/uv/install.sh | sh
```

**Remote (Windows):**
```
try { uv --version } catch { Invoke-RestMethod https://astral.sh/uv/install.ps1 | Invoke-Expression }
```

Expected: `uv --version` returns a version string (e.g., `uv 0.5.x`).

If this fails:
- **macOS:** Ensure Homebrew is installed (`which brew`). Alternatively use the curl method: `curl -LsSf https://astral.sh/uv/install.sh | sh`.
- **Linux:** Check internet connectivity. Ensure `curl` is available.
- **Windows:** Check PowerShell execution policy. Try `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned` first.

If already exists: Skip. `uv` is already installed.

#### 1.3 Verify Python 3 `[AUTO]`

Confirm Python 3 is available. `uv` can manage Python versions, but it's good to verify.

**Remote (macOS/Linux):**
```
python3 --version 2>/dev/null || python --version 2>/dev/null || echo 'NO_PYTHON'
```

**Remote (Windows):**
```
try { python --version } catch { 'NO_PYTHON' }
```

Expected: Python 3.9 or later.

If this fails: `uv` can install Python automatically. Proceed to Phase 2 — `uv run` will handle it. If `uv run` also fails on missing Python, install it: `uv python install 3.12`.

### Phase 2: Deploy the Skill

#### 2.1 Create the Skill Directory `[AUTO]`

Create the directory structure for the Nano Banana Pro 2 skill.

**Remote:**
```
mkdir -p ${WORKSPACE}/skills/nano-banana-pro-2/scripts
```

Expected: Directory tree exists at `${WORKSPACE}/skills/nano-banana-pro-2/scripts/`.

If already exists: This is safe to re-run (`mkdir -p` is idempotent).

#### 2.2 Write the Image Generation Script `[GUIDED]`

Write the `generate_image.py` script that wraps the Gemini Image API. This is the core of the skill.

The script should be copied from the source repository or written to the skill directory. The operator should have the script content available from the skill source.

**Remote:**
```
cp -r /path/to/nano-banana-pro-2/* ${WORKSPACE}/skills/nano-banana-pro-2/
```

> **Note:** The source path depends on where the skill files originate. If deploying from a repository clone, use the repo path. If the files are bundled with OpenClaw, check `${WORKSPACE}/skills/` for templates. The operator should determine the correct source.

Expected: `generate_image.py` exists in `${WORKSPACE}/skills/nano-banana-pro-2/scripts/`.

**Remote (verify):**
```
ls ${WORKSPACE}/skills/nano-banana-pro-2/scripts/
```

Expected: `generate_image.py` and supporting files (e.g., `pyproject.toml` at the skill root).

If this fails: Verify the source path. Check that the source repository is available and the files are in the expected location.

If already exists: Compare the existing script with the new version. If unchanged, skip. If different, back up the existing version as `generate_image.py.bak` and write the new one.

#### 2.3 Verify Skill Structure `[AUTO]`

Confirm the complete skill structure is in place.

**Remote (macOS/Linux):**
```
ls -la ${WORKSPACE}/skills/nano-banana-pro-2/ && ls -la ${WORKSPACE}/skills/nano-banana-pro-2/scripts/
```

**Remote (Windows):**
```
Get-ChildItem '${WORKSPACE}\skills\nano-banana-pro-2' -Recurse -Name
```

Expected: At minimum, these files exist:
- `${WORKSPACE}/skills/nano-banana-pro-2/pyproject.toml`
- `${WORKSPACE}/skills/nano-banana-pro-2/scripts/generate_image.py`

If this fails: Re-run step 2.2. Check file permissions.

### Phase 3: Test and Verify

#### 3.1 Test Text-to-Image Generation `[GUIDED]`

Run a simple text-to-image generation to confirm the full pipeline works: API key, `uv` dependency resolution, Gemini API call, and file output.

**Remote:**
```
cd ${WORKSPACE}/skills/nano-banana-pro-2 && uv run scripts/generate_image.py --prompt 'a red lobster wearing sunglasses on a beach' --filename '/tmp/test-gen.png'
```

Expected: Command exits with code 0 and `/tmp/test-gen.png` is created.

**Remote (verify output):**
```
ls -la /tmp/test-gen.png
```

Expected: File exists, size is 10KB+ (a simple image should be at least this large).

If this fails:
- **API authentication error:** Check `GEMINI_API_KEY` in `${WORKSPACE}/.env`. Verify the key has image generation access enabled in Google AI Studio.
- **`uv` dependency errors:** Run `uv sync` in the skill directory to force dependency resolution, then retry.
- **Python import errors:** Check Python version (`python3 --version`). The script requires Python 3.9+.
- **Network timeout:** The API call may take 10-30 seconds. Retry if it timed out.

#### 3.2 Test Image Editing (Optional) `[GUIDED]`

Test modifying an existing image with a text prompt. This verifies the `-i` (input image) flag works.

**Remote:**
```
cd ${WORKSPACE}/skills/nano-banana-pro-2 && uv run scripts/generate_image.py --prompt 'add a top hat' -i /tmp/test-gen.png --filename '/tmp/test-edit.png'
```

Expected: `/tmp/test-edit.png` is created with the lobster now wearing a top hat.

If this fails: Check that `/tmp/test-gen.png` exists from step 3.1. The input image must be a valid PNG/JPEG.

#### 3.3 Test Multi-Image Composition (Optional) `[GUIDED]`

Test combining multiple input images into a new composition. This verifies multi-image mode.

**Remote:**
```
cd ${WORKSPACE}/skills/nano-banana-pro-2 && uv run scripts/generate_image.py --prompt 'combine these into a collage' -i /tmp/test-gen.png -i /tmp/test-edit.png --filename '/tmp/test-composite.png'
```

Expected: `/tmp/test-composite.png` is created combining both input images.

If this fails: Reduce the number of input images. Check that all input files exist and are valid images. Maximum is 14 input images.

#### 3.4 Clean Up Test Files `[AUTO]`

Remove the test output files.

**Remote (macOS/Linux):**
```
rm -f /tmp/test-gen.png /tmp/test-edit.png /tmp/test-composite.png
```

**Remote (Windows):**
```
Remove-Item -Path 'C:\Users\*\AppData\Local\Temp\test-gen.png','C:\Users\*\AppData\Local\Temp\test-edit.png','C:\Users\*\AppData\Local\Temp\test-composite.png' -ErrorAction SilentlyContinue
```

Expected: Test files removed.

### Phase 4: Post-Deploy

#### 4.1 Update Client Profile `[AUTO]`

Update `clients/<name>.md` with:
- Image generation deployed (Gemini 3 Pro / Nano Banana Pro 2)
- Skill location: `${WORKSPACE}/skills/nano-banana-pro-2/`
- Default resolution configured
- Any issues encountered

#### 4.2 Note Follow-Ups `[AUTO]`

After image generation is deployed, related skills that pair well:
- `deploy-video-gen.md` — Adds video generation (Veo 3) alongside image generation
- `deploy-video-analysis.md` — Adds analysis of images and video

## Verification

Run these checks to confirm everything is working:

**Remote (all platforms):**
```
cd ${WORKSPACE}/skills/nano-banana-pro-2 && uv run scripts/generate_image.py --prompt "test image: a simple red circle on white background" --filename "/tmp/test-verify.png"
```

**Remote (macOS/Linux — check output):**
```
ls -la /tmp/test-verify.png
```

**Remote (Windows — check output):**
```
Get-ChildItem "$env:TEMP\test-verify.png"
```

Expected:
- Command exits with code 0
- Output file exists and is a valid PNG
- File size is 10KB+ for a simple image

**Remote (clean up):**
```
rm -f /tmp/test-verify.png
```

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| API error / authentication failure | `GEMINI_API_KEY` is invalid or missing | Verify the key in `${WORKSPACE}/.env`. Check that the key has Imagen/image generation access enabled in Google AI Studio. |
| `uv` not found | Package manager not installed | macOS: `brew install uv`. Linux: `curl -LsSf https://astral.sh/uv/install.sh \| sh`. Windows: `irm https://astral.sh/uv/install.ps1 \| iex`. |
| Resolution too high / API rejection | 4K resolution not supported for certain prompts | Try `--resolution 2K` or `--resolution 1K`. Some prompts work better at lower resolutions. |
| Python errors on first run | Dependencies not resolved | `uv` handles this automatically via `pyproject.toml`. If issues persist, run `uv sync` in the skill directory. |
| Output image is blank or garbled | Prompt too vague or conflicting | Try a more specific prompt. Shorter, concrete descriptions produce better results than long abstract ones. |
| Multi-image composition fails | Too many input images or files too large | Reduce to fewer input images (max 14). Resize large inputs before passing them. |
| `uv run` fails with "no Python found" | Python not installed on client | Run `uv python install 3.12` to let uv manage its own Python. |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Verify Gemini API Key | Prompt for key if missing | Prompt for key if missing | Prompt for key if missing |
| 1.2 Install uv | Execute silently | Confirm before install | Confirm the install command |
| 1.3 Verify Python 3 | Execute silently | Execute silently | Show version, confirm |
| 2.1 Create skill directory | Execute silently | Execute silently | Confirm directory creation |
| 2.2 Write image gen script | Execute silently | Confirm before writing | Show file list, confirm each |
| 3.1 Test text-to-image | Execute, report result | Execute, show output | Confirm before running |
| 3.2 Test image editing | Skip unless client requests | Offer to test | Offer to test |
| 3.3 Test multi-image | Skip unless client requests | Offer to test | Offer to test |
| 3.4 Clean up test files | Execute silently | Execute silently | Confirm before deleting |
| 4.1 Update client profile | Execute silently | Execute silently | Show changes, confirm |

## Dependencies

- **Depends on:** `openclaw/install.md` (OpenClaw must be installed and running)
- **Required by:** Nothing — this is a standalone creative capability
- **Enhanced by:** `deploy-video-gen.md` (video generation), `deploy-video-analysis.md` (image/video analysis)
