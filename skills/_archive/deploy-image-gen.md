# Deploy Image Generation

## Purpose

Install image generation capabilities via Google Gemini 3 Pro Image API (Nano Banana). Supports text-to-image, image editing, and multi-image composition for thumbnails, social posts, and visual assets.

**When to use:** When the user needs on-demand image generation.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "which uv 2>/dev/null"
cmd --session <id> --command "echo $GEMINI_API_KEY | head -c 8"
```
- Is `uv` installed? (Required for the default Gemini integration)
- Is GEMINI_API_KEY configured?
- Does the client need image generation at all?

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Alternative | When to Adapt |
|-----------|---------|-------------|---------------|
| Image API | Google Gemini 3 Pro (Nano Banana) | OpenAI DALL-E 3, Stability AI, Midjourney API | Client prefers a different image provider or doesn't have Gemini access |
| Runtime | `uv` (Python) | pip, conda | Client uses a different Python package manager |
| Default resolution | 1K | 2K or 4K | Client typically needs higher resolution output |
| Output path | Timestamped files in working directory | Custom output directory | Client wants organized asset storage |

## Prerequisites

- `GEMINI_API_KEY` configured in `.env`
- `uv` Python package manager: `brew install uv`
- Python 3 available

## What Gets Installed

### Nano Banana Pro 2 Skill (`skills/nano-banana-pro-2/`)

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

### 1. Verify GEMINI_API_KEY

```
cmd --session <id> --command "grep GEMINI_API_KEY ~/clawd/.env"
```

If missing, add it:

```
cmd --session <id> --command "echo 'GEMINI_API_KEY=<your-key>' >> ~/clawd/.env"
```

### 2. Install uv

```
cmd --session <id> --command "which uv || brew install uv"
```

### 3. Install the Nano Banana Pro 2 Skill

```
cmd --session <id> --command "cp -r /path/to/nano-banana-pro-2 ~/clawd/skills/nano-banana-pro-2"
```

Verify the structure:

```
cmd --session <id> --command "ls ~/clawd/skills/nano-banana-pro-2/scripts/"
```

Expected: `generate_image.py` and supporting files.

### 4. Test Text-to-Image Generation

```
cmd --session <id> --command "cd ~/clawd/skills/nano-banana-pro-2 && uv run scripts/generate_image.py --prompt 'a red lobster wearing sunglasses on a beach' --filename '/tmp/test-gen.png'"
```

### 5. Verify Output

```
cmd --session <id> --command "ls -la /tmp/test-gen.png"
```

The file should exist and be a valid PNG. Open it to confirm the image matches the prompt.

### 6. Test Image Editing (Optional)

```
cmd --session <id> --command "cd ~/clawd/skills/nano-banana-pro-2 && uv run scripts/generate_image.py --prompt 'add a top hat' -i /tmp/test-gen.png --filename '/tmp/test-edit.png'"
```

### 7. Test Multi-Image Composition (Optional)

```
cmd --session <id> --command "cd ~/clawd/skills/nano-banana-pro-2 && uv run scripts/generate_image.py --prompt 'combine these into a collage' -i /tmp/test-gen.png -i /tmp/test-edit.png --filename '/tmp/test-composite.png'"
```

## Verification

Run the full test suite:

```
cd ~/clawd/skills/nano-banana-pro-2
uv run scripts/generate_image.py --prompt "test image: a simple red circle on white background" --filename "/tmp/test-verify.png"
ls -la /tmp/test-verify.png
```

Expected:
- Command exits with code 0
- Output file exists and is a valid PNG
- File size is reasonable (10KB+ for a simple image)

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| API error / authentication failure | `GEMINI_API_KEY` is invalid or missing | Verify the key in `.env`. Check that the key has Imagen/image generation access enabled in the Google AI console. |
| `uv` not found | Package manager not installed | Run `brew install uv` (macOS) or check [uv installation docs](https://docs.astral.sh/uv/) for other platforms. |
| Resolution too high / API rejection | 4K resolution not supported for certain prompts | Try `--resolution 2K` or `--resolution 1K`. Some prompts work better at lower resolutions. |
| Python errors on first run | Dependencies not resolved | `uv` handles this automatically via `pyproject.toml`. If issues persist, try `uv sync` in the skill directory. |
| Output image is blank or garbled | Prompt too vague or conflicting | Try a more specific prompt. Shorter, concrete descriptions produce better results than long abstract ones. |
| Multi-image composition fails | Too many input images or files too large | Reduce to fewer input images (max 14). Resize large inputs before passing them. |

## Dependencies

- **Requires:** `GEMINI_API_KEY` in `.env`
- **Independent of:** All other deploy skills
