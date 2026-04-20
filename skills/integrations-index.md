# Integration Skills Catalog

This file is the canonical index of **integration-style skills** in `openclaw-base/skills/`. These skills connect the agent to third-party services (APIs, OAuth providers, browser-automation targets) and are distinct from the memory-option research files in `memory-options/` and the platform-primitive skills like `deploy-memory.md` or `deploy-identity.md`.

**Files stay flat in `skills/`** (not in a subdirectory) for backwards compatibility with existing client deployments and older runbook references. Every file listed below has YAML frontmatter describing its service, auth model, and ClawHub alternative — use that frontmatter as the source of truth; this index is generated reading material for humans and audit agents.

All paths are absolute under `skills/`.

---

## Productivity

| File | Service | Subcategory | Auth | Status | ClawHub alt | Last checked |
|------|---------|-------------|------|--------|-------------|--------------|
| `deploy-asana.md` | Asana | productivity | personal-access-token | active | `asana-api` | 2026-04-20 |
| `deploy-google-workspace.md` | Google Workspace (Gmail/Calendar/Drive) | productivity | oauth | active | `@steipete/gog` | 2026-04-20 |

## CRM

| File | Service | Subcategory | Auth | Status | ClawHub alt | Last checked |
|------|---------|-------------|------|--------|-------------|--------------|
| `deploy-newsletter-crm.md` | Beehiiv + HubSpot | crm | api-key | active | `beehiiv` | 2026-04-20 |

## Meeting intelligence

| File | Service | Subcategory | Auth | Status | ClawHub alt | Last checked |
|------|---------|-------------|------|--------|-------------|--------------|
| `deploy-fathom-pipeline.md` | Fathom | meeting-intelligence | api-key | active | `fathom-api` | 2026-04-20 |

## Social posting

| File | Service | Subcategory | Auth | Status | ClawHub alt | Last checked |
|------|---------|-------------|------|--------|-------------|--------------|
| `deploy-tiktok-account-setup.md` | TikTok | social-posting | scraping | active | `tiktok-uploader` | 2026-04-20 |
| `deploy-tiktok-warmup.md` | TikTok | social-posting | scraping | active | none | 2026-04-20 |
| `deploy-tiktok-compliance.md` | TikTok | social-posting | scraping | active | none | 2026-04-20 |
| `deploy-tiktok-posting.md` | TikTok | social-posting | scraping | active | `tiktok-uploader` | 2026-04-20 |
| `deploy-youtube-posting.md` | YouTube Data API v3 | social-posting | oauth | active | `upload-post` | 2026-04-20 |

## Content generation

| File | Service | Subcategory | Auth | Status | ClawHub alt | Last checked |
|------|---------|-------------|------|--------|-------------|--------------|
| `deploy-content-pipeline.md` | SQLite + Telegram | content-gen | api-key | active | none | 2026-04-20 |
| `deploy-image-gen.md` | Gemini API (Nano Banana Pro) | content-gen | api-key | active | `nano-banana-pro` | 2026-04-20 |
| `deploy-video-gen.md` | Gemini API (Veo 3) | content-gen | api-key | active | `video-generation-minimax` | 2026-04-20 |
| `deploy-video-analysis.md` | Gemini API | content-gen | api-key | active | none | 2026-04-20 |
| `deploy-video-research.md` | Telegram + Gemini API | content-gen | api-key | active | none | 2026-04-20 |
| `deploy-video-pipeline.md` | Slack + Asana + X/Twitter | content-gen | api-key | active | none | 2026-04-20 |
| `deploy-humanizer.md` | n/a (local LLM pass) | content-gen | api-key | active | `@biostartechnology/humanizer` | 2026-04-20 |

---

## How to add a new integration

1. **Pick a filename** — follow the existing `deploy-<service-or-flow>.md` convention. Keep it flat in `skills/`.
2. **Write the skill body** using the existing deploy-* files as style references (Compatibility → Purpose → Variables → Phases). See `_authoring/_skill-authoring-guide.md`.
3. **Prepend YAML frontmatter** — mirror the shape of `memory-options/_template.md` but adapted to the integration context. Required keys:
   - `name` — matches the H1 of the file
   - `category: "integration"`
   - `subcategory` — one of: `productivity | crm | content-gen | social-posting | meeting-intelligence | communication`
   - `third_party_service` — human-readable service name
   - `auth_type` — one of: `api-key | oauth | personal-access-token | scraping | app-password`
   - `openclaw_version_required`, `version_last_verified`, `last_checked`
   - `maintenance_status: "active" | "dormant" | "deprecated" | "preview" | "unverified"`
   - `model_env_var` — the per-skill model override env var (see `_authoring/_deploy-common.md` "Per-skill env-var overrides"), or `"n/a"` if the skill makes no LLM calls
   - `clawhub_alternative` — slug from `audit/_baseline.md` §B if one exists, else `"none"`
   - `cost_tier`, `privacy_tier`, `requires`, `docs_urls`
4. **Add a row to this index** in the correct subcategory section.
5. **Update `audit/_baseline.md` §B** if you found a new ClawHub alternative worth tracking.

## For audit agents

When running a skill audit or writing a new one:

- **Grade each integration skill against `audit/_baseline.md`.** Every frontmatter claim (version, clawhub alt, last_checked) should be verifiable against that doc or a live probe.
- **Check `clawhub_alternative`** — if the ClawHub alternative is more popular/maintained than our deploy-* runbook, the skill may be a candidate for retirement in favor of `clawhub install <slug>`. See `_baseline.md` §B "Implications".
- **`maintenance_status: "active"` is a claim, not a default.** If `last_checked` is older than 90 days, treat `active` as stale and re-verify.
- **`model_env_var` = `"n/a"`** means the skill makes no LLM calls in its own execution path. Skills with an env var route their LLM calls through it — verify the `_deploy-common.md` "Per-skill env-var overrides" pattern is actually being used in the skill body.
- **Platform-primitive skills** (`deploy-memory.md`, `deploy-identity.md`, `deploy-db-backups.md`, `deploy-telegram-forum.md`, etc.) are intentionally NOT in this index — they configure the OpenClaw workspace itself rather than integrating a named third-party service. If a skill's job is "install a local system", it's a platform primitive, not an integration.
- **`deploy-content-pipeline.md` is an edge case** — it provisions a local SQLite DB and a Telegram group layout. It's listed here because downstream integrations (TikTok, YouTube, video-research) depend on it and because it does interact with Telegram. If you reclassify it, also move every skill that `requires:` it.
