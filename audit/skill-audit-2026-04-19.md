# Skill Audit — openclaw-base

**Started:** 2026-04-19
**Auditor:** Claude (direct review per `feedback_no_subagent_audits.md`; subagents OK for bulk research only)
**Baseline:** `audit/_baseline.md` (built first, this audit grades against it)
**Adversarial review (planned):** Codex pass after Claude's pass to catch misses (output → `audit/skill-audit-2026-04-19-codex-review.md`)

## How To Read This Ledger

Every skill below has a row with these columns:

| Column | Meaning |
|--------|---------|
| **#** | Audit ID |
| **Skill** | File path |
| **Score** | GREEN / YELLOW / ORANGE / RED (see legend) |
| **Format** | `deploy-X` (our runbook format, operator-invoked) or `SKILL.md` (canonical OpenClaw agent-loadable) |
| **ClawHub Eqv** | Link to popular ClawHub skill that does the same thing, or `none` |
| **Bundled Eqv** | Name of bundled OpenClaw skill (in 4C's 43/52 ready) that does the same thing, or `none` |
| **Evidence** | Specific lines from the skill + § refs from `_baseline.md` |
| **Action** | KEEP / REWRITE / WRAP-CLAWHUB / REPLACE-WITH-CLAWHUB / REPLACE-WITH-BUNDLED / DELETE |

## Score Legend

| Score | Meaning |
|-------|---------|
| GREEN | Matches current OpenClaw 2026.4.15 conventions, models are current, accurate |
| YELLOW | Mostly correct; ≤30 min of fixes needed (model IDs, minor API drift) |
| ORANGE | Major rewrite needed; underlying approach sound but implementation stale |
| RED | Concept obsolete OR superseded by built-in/ClawHub OR fundamentally broken |

## Action Legend

| Action | What happens |
|--------|--------------|
| **KEEP** | File stays as-is (GREEN scores) |
| **REWRITE** | Add `<!-- TODO -->` block at top, plan rewrite next pass (YELLOW or ORANGE) |
| **WRAP-CLAWHUB** | Slim down our skill to just install + configure a ClawHub skill |
| **REPLACE-WITH-CLAWHUB** | DELETE our file, add ClawHub install instructions to a "replacements" section below |
| **REPLACE-WITH-BUNDLED** | DELETE our file, document which bundled skill to use instead |
| **DELETE** | Remove the file entirely (RED with no replacement) |

Per the operator: "for the ones that are bad, just throw them out, delete them" — no soft-archive for obsolete content.

## Method (per skill)

1. Read the skill file end-to-end (Read tool — visible in transcript)
2. Quick targeted research if needed (WebSearch for that skill's domain, ≤3 queries)
3. Compare to baseline (`_baseline.md` § citations required in Evidence)
4. Check ClawHub for equivalent (`_baseline.md` §B has the master comparison)
5. Check 4C bundled skills (`_baseline.md` §4C.5)
6. Score + write row + (if DELETE) execute deletion + (if REWRITE/REPLACE) write detail block at bottom

## Baseline context to pre-load before any scoring

- Live 4C runs **OpenClaw 2026.4.15** (§4C.1 in `_baseline.md`).
- Many skill files carry a "Compatibility" header dated **2026.2.26** (Feb 26, 2026 — about **8 weeks** before today, NOT "years"). The skills themselves are recent; they just pre-date `2026.3.x` and `2026.4.x` which shipped several features the skills' "Blocked Commands" lists say don't exist (e.g., `openclaw cron`, `openclaw channels`, `openclaw memory`).
- Do NOT conflate the header version string `2026.2.26` with "years old." That's ~8 weeks. Reframe any row that says the skill is "stale by years" — it's weeks, with specific claims now obsolete.
- **Correct version framing:** "written against 2026.2.26; some `Blocked Commands` claims are resolved in 2026.4.15 — verify against live 4C probes in §4C."

## Replacements Tracking

When a skill is REPLACE-WITH-CLAWHUB or REPLACE-WITH-BUNDLED, log it here so we have an install playbook:

(pending the operator's manual review of ClawHub alternatives per 2026-04-19 directive)

## Sweep Actions Log (2026-04-19)

Mechanical changes applied across top-level `skills/` files after the adversarial review caught systemic issues:

### P1 Model-ID sweep (retiring Anthropic snapshots → stable aliases)

Rationale: baseline §X shows `claude-sonnet-4-20250514`, `claude-opus-4-20250514`, `claude-haiku-*-20250514` retire **2026-06-15**. Every hardcoded use would break on that date.

| File | Before (refs) | After (refs) |
|------|---------------|--------------|
| `deploy-action-items.md` | 4 | 0 |
| `deploy-meeting-prep.md` | 2 | 0 |
| `deploy-platform-health.md` | 1 | 0 |
| `deploy-security-council-v2.md` | 4 | 0 |
| `deploy-security-council.md` | 1 | 0 |
| `deploy-transcript-pipeline.md` | 2 | 0 |

**Replacement mapping:** `claude-sonnet-4-20250514` → `claude-sonnet-4-6` (stable alias §W); `claude-opus-4-20250514` → `claude-opus-4-7`; `claude-haiku-4-20250514` → `claude-haiku-4-5`. Plus backstop sweeps for `claude-3-*-*` legacy snapshots.

Backup of pre-sweep state: `/tmp/openclaw-base-backup-2026-04-19/` (48 files).

### Client-identifier sweep (moderate-leakage files)

Rationale: adversarial review (`skill-audit-2026-04-19-codex-review.md`) identified systemic leakage of {{FAMILY_NAME}}/Rel identifiers in "base" skills. Moderate-leakage files (examples only, not structural) scrubbed mechanically:

| File | {{FAMILY_NAME}} refs before | After |
|------|-------------------|-------|
| `deploy-linear-integration.md` | 3 | 0 |
| `deploy-action-items.md` | 6 | 0 |
| `deploy-meeting-prep.md` | 2 | 0 |
| `deploy-notion-workspace.md` | 1 | 0 |
| `deploy-security-council-v2.md` | 1 | 0 |

**Replacement mapping:**
- `edge@{{COMPANY_DOMAIN}}` → `<AGENT_EMAIL>`
- `{{PRINCIPAL_EMAIL}}` → `<CLIENT_EMAIL>`
- `@{{COMPANY_DOMAIN}}` → `@<INTERNAL_DOMAIN>`
- `{{COMPANY_DOMAIN}}` → `<INTERNAL_DOMAIN>`
- `{{PRINCIPAL_CHAT_ID}}` → `<TELEGRAM_USER_ID>`
- `<TELEGRAM_GROUP_ID>` → `<TELEGRAM_GROUP_ID>`
- `"{{PRINCIPAL_NAME}}"` → `"<CLIENT_NAME>"` (in example/mock data)

### Saturated-leakage files awaiting the operator's decision

Structural {{FAMILY_NAME}} flavoring that a mechanical sweep can't fix — Purpose sections, regex patterns, and allowlists are engagement-specific:

- `deploy-himalaya-email.md` (row 28) — recommend DELETE (reference-snapshot copy preserved) or heavy REWRITE
- `deploy-github-cicd.md` (row 27) — recommend DELETE or full REWRITE to generics or REPLACE-WITH-CLAWHUB for `@steipete/github`

### Compatibility header rewrites (ORANGE → WORKING)

Previous "BLOCKED — N nonexistent commands" warnings were stale against 2026.4.15 (live-verified 4C §3/§4C.7). Header rewritten to mark WORKING + flag remaining architectural concerns:

- `_authoring/_deploy-common.md` — full rewrite (649 → ~205 lines, OAC-stripped, Codex-default)
- `deploy-platform-health.md`
- `deploy-health-monitoring.md` (compat header fixed, but ledger row 46 still flagged RED for architectural fan-out anti-pattern per adversarial review §O)
- `deploy-earnings.md`
- `deploy-food-journal.md`
- `deploy-personal-crm.md`
- `deploy-knowledge-base.md`
- `deploy-advisory-council.md`

### Deletions executed (first pass)

- `install/setup-macos.md`, `install/setup-windows.md`, `install/setup-ubuntu-gce.md` (OAC-enrollment, not OpenClaw install)
- `skills/deploy-personal-crm-v2.md`, `deploy-daily-briefing-v2.md`, `deploy-urgent-email-v2.md` (engagement-specific duplicates)
- `skills/deploy-git-autosync.md` (superseded by v2)

### reference purge (second pass, per the operator 2026-04-19)

the operator directive: "We should not be using anything from the reference engagement Ventures as the exact path. That is a custom-made resource and it should just be something on the side for reference when we're doing custom-made skills for offices. So all the {{FAMILY_NAME}} Ventures stuff, ignore it."

All reference-originated files deleted from `skills/`. Reference copies preserved in `skills/_enterprise-v2-reference/` for when the operator wants to deploy a custom office/VC setup.

| Row | File | Prior ledger state | Final state |
|-----|------|-------------------|-------------|
| 13 | `deploy-security-council.md` | ORANGE — DELETE-after-merge | **DELETED**. Keychain-incompatibility warning (the valuable L21-38 institutional lesson) merged into `_authoring/_deploy-common.md` "OpenClaw General" gotchas table (new "Secret storage" section) |
| 14 | `deploy-security-council-v2.md` | YELLOW — REWRITE | **DELETED** (reference-originated, identical to reference copy) |
| 21 | `deploy-action-items.md` | YELLOW — REWRITE | **DELETED** (reference-originated; L45 chat ID was the reference snapshot's) |
| 22 | `deploy-meeting-prep.md` | ORANGE (P1 DEADLINE) — REWRITE | **DELETED** — P1 deadline moot, file gone from base |
| 23 | `deploy-transcript-pipeline.md` | YELLOW — REWRITE | **DELETED** (reference-originated) |
| 25 | `deploy-notion-workspace.md` | YELLOW — REWRITE | **DELETED** (reference-originated, skills/ was subset of reference) |
| 26 | `deploy-linear-integration.md` | YELLOW — REWRITE | **DELETED** (reference-originated) |
| 27 | `deploy-github-cicd.md` | ORANGE (engagement-saturated) | **DELETED** (reference-originated, "for {{COMPANY_NAME}} projects") |
| 28 | `deploy-himalaya-email.md` | ORANGE (engagement-saturated) | **DELETED** (reference-originated, references "{{AGENT_NAME}} AI" structurally) |
| 45 | `deploy-git-autosync-v2.md` | YELLOW — REWRITE | **DELETED** (reference-originated even though content was generic — per the operator's strict "don't use anything from the reference engagement" rule) |

After second pass: **38 top-level skill files remain in `skills/`**, all generic (no {{FAMILY_NAME}} identifiers remaining). Reference folder unchanged (24 files).

### Row 52 polish applied (diagnose-cron.md)

Per adversarial review verdict (KEEP — 15-min polish only):
- L220 version bumped from `2026.3.13` → `2026.4.15+` (with live-verified note)
- L216 Rel HQ chat ID `<TELEGRAM_GROUP_ID>` → `<CLIENT_CHAT_ID>` placeholder
- L217 Sprint topic `156` → `<CLIENT_TOPIC_ID>` placeholder

Skill is now clean. Row 52 effective state: **GREEN / DONE**.

### Rewrites completed 2026-04-20 (session 2)

Structural / full rewrites:
| File | Row | Before | After |
|------|-----|--------|-------|
| `_authoring/_deploy-common.md` | 57 | ORANGE, 649 lines, OAC-heavy | ~230 lines, OAC-stripped, Codex-default, Keychain lesson preserved |
| `_authoring/_skill-review.md` | 59 | ORANGE, subagent-delegation pattern (L183-188) | ~175 lines, harness-pattern documented, subagent anti-delegation rule embedded |
| `_authoring/_skill-authoring-guide.md` | 58 | YELLOW, Opus-anchored, no SKILL.md section | targeted edits: Opus→model-agnostic, hardcoded-snapshot example replaced with "defer to agent default" note, anti-pattern #11 added (placeholder-hand-edit), Two-Formats section added |
| `install/openclaw-install.md` | 53 | ORANGE, 635 lines with 270-line Phase 1 OAC enrollment | ~170 lines, zero OAC content, clean generic install |
| `deploy-health-monitoring.md` | 46 | RED (architectural fan-out) | ~170 lines, collapsed 3 cron jobs → single daily heartbeat per §L/§O community pattern, bash-first with LLM-on-alert escalation |
| `deploy-advisory-council.md` body | 12 | header rewritten but body hardcoded `ANTHROPIC_API_KEY` | 6 Anthropic refs stripped; model routing defers to `agents.defaults.model`; per-skill override via `COUNCIL_PERSONA_MODEL`/`COUNCIL_SYNTHESIS_MODEL` env |

Compat-header rewrites applied earlier (ORANGE → WORKING):
- `deploy-platform-health.md`, `deploy-earnings.md`, `deploy-food-journal.md`, `deploy-personal-crm.md`, `deploy-knowledge-base.md`, `deploy-asana.md`, `deploy-newsletter-crm.md`, `deploy-fathom-pipeline.md`, `deploy-video-pipeline.md`

Batch version bumps (YELLOW) — 23 files from `2026.2.26` → `2026.4.15+ (bumped 2026-04-20)`:
`deploy-content-pipeline`, `deploy-daily-briefing`, `deploy-db-backups`, `deploy-google-workspace`, `deploy-humanizer`, `deploy-identity`, `deploy-image-gen`, `deploy-memory`, `deploy-messaging-setup`, `deploy-model-tracking`, `deploy-prompt-guide`, `deploy-security-safety`, `deploy-social-tracking`, `deploy-telegram-forum`, `deploy-tiktok-account-setup`, `deploy-tiktok-compliance`, `deploy-tiktok-posting`, `deploy-tiktok-warmup`, `deploy-urgent-email`, `deploy-video-analysis`, `deploy-video-gen`, `deploy-video-research`, `deploy-youtube-posting`

### YELLOW polish sweep (2026-04-20, session 3 — parallel subagents + harness)

Three subagents dispatched in parallel. Findings verified by Claude.

**Subagent A — reference-snapshot snapshot notice (23 files modified)**
Added standardized `> **📌 REFERENCE SNAPSHOT — NOT FOR DEFAULT DEPLOYMENT**` block to the top of every file in `skills/_enterprise-v2-reference/` (except README.md). The notice explains the snapshot came from the reference engagement, warns against using as a default, flags it may contain stale `2026.2.26` Blocked Commands claims + Claude-Opus routing. This makes accidental "install from reference folder" less dangerous.

**Subagent B — deploy-prompt-guide.md Opus refresh (3 edits)**
Per adversarial review Row 11:
- L80 Adaptation Points: "Claude Opus 4.6" target → "Model-agnostic by default"
- L140 Primary Model table: "Claude Opus 4.6 (default)" → "Claude Opus 4.7 (default) / Opus 4.6 (legacy)"
- L247-257 Model-Specific section: renamed to "Claude Opus 4.7 (current flagship) + Opus 4.6 (legacy)"; added adaptive-thinking bullet (reasoning.effort levels, `xhigh` per §8); added tokenizer caution bullet (1M tokens ≈ 555k words for 4.7 vs 750k for Sonnet 4.6)
- Universal Principles + DeepSeek/GPT/Gemini/Other sections preserved untouched

**Subagent C — OAC strip from YELLOW skills (37 files processed, 32 modified)**
- 32 `## How Commands Are Sent` sections replaced with: `Standard protocol — see \`_authoring/_deploy-common.md\`. \`Remote:\` commands run on the target machine; \`Operator:\` commands run locally.`
- ~55 OAC-specific variable rows removed across files (`${BEARER}`, `${SESSION_ID}`, `${DEVICE_ID}`, `${ENROLLMENT_KEY}`, `${CLIENT_ID}` where it was OAC-flavored)
- 3 files had no such section (skipped cleanly): `deploy-mission-control.md`, `token-optimization.md`, `diagnose-cron.md`
- 2 files already used the standard pointer (no-op): `deploy-advisory-council.md`, `deploy-health-monitoring.md`
- Subagent spot-checked 3 random files post-sweep — all clean

**Subagent C flagged: `oauth-reauth.md` is OAC-dependent in the body**
The skill's core mechanism is `POST /api/devices/${DEVICE_ID}/exec` — it was always OAC-operator-flavored. Native `openclaw models auth login` handles re-auth on any install without needing a skill.
→ **DELETED from `skills/`** (2026-04-20). Users who need re-auth just run `openclaw models auth login --provider <x>` directly.

**Authoring guide template cleanup (Claude direct, 2 edits)**
`_authoring/_skill-authoring-guide.md` had 2 template sections still showing the old OAC `POST /api/sessions/${SESSION_ID}/commands` pattern (in the anatomy example and the new-skill template). Both replaced with the generic pointer.

### Final verification (2026-04-20, post-polish)

Grep across `skills/deploy-*.md` + `install/openclaw-install.md` for red-flag patterns returned **zero true positives**:
- `openclawinstall\.net` / OAC-specific endpoints: **0 matches** in top-level skills, 0 in install
- `ANTHROPIC_API_KEY` hardcoded: **0 matches** (L9 + L54 of advisory-council are "we removed this" notes, not leaks)
- `sk-ant-` hardcoded: **0 matches** (`deploy-security-safety.md` L320 is a DETECTOR regex, not a key)
- Retiring `claude-(sonnet|opus|haiku)-[34]-20250514` snapshots: **0 matches** in top-level
- `2026.2.26` version headers: **0 matches** in top-level
- {{FAMILY_NAME}} identifiers (`{{COMPANY_SLUG}}`, `{{PRINCIPAL_NAME}}`, `{{PRINCIPAL_CHAT_ID}}`, `<TELEGRAM_GROUP_ID>`): **0 matches** in top-level skills

reference-snapshot folder (`_enterprise-v2-reference/`) and archive (`_archive/`) intentionally unchanged — those are reference snapshots for custom office deployments.

### Gaps created by the reference purge (future work)

`skills/` no longer has default implementations for these capabilities. If they're needed:
- **Linear integration** → install ClawHub `linear-api` (managed-OAuth) or `linear-autopilot`
- **Notion workspace** → install `@steipete/notion` (78k dl)
- **GitHub CI/CD** → install `@steipete/github` (160k dl)
- **Meeting prep** → install ClawHub `meeting-prep` or `ai-meeting-prep`
- **Transcript ingestion** → install `fathom-api` / `video-transcript-downloader` or adapt from reference folder
- **Himalaya email** (macOS-only fast email via IMAP) → adapt from reference folder if needed; `@steipete/gog` is the standard Gmail path
- **Action items extraction** → adapt from reference folder for custom office deployment
- **Hourly git autosync** → no ClawHub equivalent (per §B); rebuild generically if needed, or adapt from reference copy
- **Security council** (nightly AI review) → adapt from reference folder for custom office deployment; or install `@-/security-reviewer`

## Per-Skill Detail Blocks

Non-GREEN findings get a detail block at the bottom of this file (after the tables) with format:

```
### deploy-XXX.md — <SCORE>

**Lines reviewed:** <range>
**Evidence:**
- <specific finding> (cite §X from _baseline.md)
- <specific finding>

**Recommendation:**
<concrete next step>

**ClawHub alternative considered:** <name + URL or "n/a">
**Bundled alternative considered:** <name or "n/a">
```

(empty — populated as audit progresses)

---

# Audit Ledger

## Group 1: Authoring meta-files (read first — defines the rules of the game)

| # | Skill | Score | Format | ClawHub Eqv | Bundled Eqv | Evidence | Action |
|---|-------|-------|--------|-------------|-------------|----------|--------|
| 57 | `_authoring/_deploy-common.md` | ORANGE | meta | none | none | Line 9 Opus 4.6 hardcoded (§BB); L582-586 falsely claims openclaw cron/telegram/messaging don't exist (§3, §4C.3, live probe refutes); OAC-heavy; good parts = Phase 0.5 compatibility-check pattern, file-transfer base64 standard, Platform Gotchas table | REWRITE |
| 58 | `_authoring/_skill-authoring-guide.md` | YELLOW | meta | none | none | Line 9 Opus 4.6 operator (§BB); L220 example uses `claude-sonnet-4-20250514` which retires 2026-06-15 (§X); Anti-Patterns + Cross-Platform table excellent; template uses `# Deploy [Name]` H1 not canonical `name:` YAML frontmatter (§6) | REWRITE |
| 59 | `_authoring/_skill-review.md` | ORANGE | meta | `skill-vetter` (different purpose) | none | L106 Opus 4.6 (§BB); L183-188 "Launch 3 agents in parallel" violates feedback_no_subagent_audits.md; L23-34 required-sections list is OAC-specific, doesn't map to canonical SKILL.md (§6); 3-lens concept (Format/Accuracy/Operability) is conceptually solid | REWRITE |

## Group 2: Install runbooks

| # | Skill | Score | Format | ClawHub Eqv | Bundled Eqv | Evidence | Action |
|---|-------|-------|--------|-------------|-------------|----------|--------|
| 53 | `install/openclaw-install.md` | ORANGE | deploy-X | none | none | Conflates OAC enrollment (Phase 1, L159-430) with OpenClaw install (Phase 2, L433-527). Phase 2 install steps mostly correct per §1 (`curl -fsSL https://openclaw.ai/install.sh \| bash` on L443). L503 `openclaw status` — NOT a documented subcommand on 2026.4.15 (§4C.7 shows `openclaw health` + `openclaw gateway status`). L634 references archived setup paths. | REWRITE (strip OAC Phase 1; fix status→health; add SKILL.md format note) |
| 54 | `install/setup-macos.md` | RED | deploy-X | none | none | Entire file is OAC agent enrollment (Session mode + LaunchAgent plist for `~/.openagent/agent/`), NOT OpenClaw install. Content belongs in rel-agent/OAC repo. Per baseline §1, installing OpenClaw on macOS = one command (`curl -fsSL https://openclaw.ai/install.sh \| bash`). This file teaches a completely different thing. | DELETE |
| 55 | `install/setup-windows.md` | RED | deploy-X | none | none | Same pattern as #54 — OAC enrollment (`irm \| iex` one-liner fetches OAC agent, not OpenClaw). Entire content is OAC-specific. Doesn't teach OpenClaw Windows install. | DELETE |
| 56 | `install/setup-ubuntu-gce.md` | RED | deploy-X | none | none | Same OAC-enrollment pattern + GCE-specific concerns. L43-44 installs Node 20 via NodeSource — but baseline §1 says OpenClaw requires **Node 22.14+** (Node 24 recommended). So even the Node step is wrong for OpenClaw install. Plus 100% of the agent/enrollment content is OAC, not OpenClaw. | DELETE |

## Group 3: Foundation skills (broadest impact)

| # | Skill | Score | Format | ClawHub Eqv | Bundled Eqv | Evidence | Action |
|---|-------|-------|--------|-------------|-------------|----------|--------|
| 1 | `deploy-identity.md` | YELLOW | deploy-X | `molt-identity`, `soul-md`, `soul-framework` | none | L4-7 Compatibility header says 2026.2.26 (stale; 4C is 2026.4.15 per §4C.1); L6 claims `openclaw restart`/`status` blocked — `status` not a top-level cmd per §4C.7 (use `openclaw health`) but `restart` unverified; L22-31 OAC-specific "How Commands Are Sent"; L87-93 Adaptation Points good; L104-105 writes IDENTITY.md + SOUL.md to `${WORKSPACE}/` ✅ matches §5 | REWRITE |
| 2 | `deploy-security-safety.md` | YELLOW | deploy-X | `security-reviewer`, `council-of-the-wise` (diff scope) | none | L4 says 2026.2.26 stale vs §4C.1; L6 falsely claims `openclaw cron add/list/status/update` blocked — LIVE §4C.3 proves these ARE available; files-first approach (content-sanitizer.js, secret-redaction.js, pre-commit hook, AGENTS.md approval gates) is version-proof and sound; Phase 5 can now use native `openclaw cron` | REWRITE |
| 3 | `deploy-messaging-setup.md` | YELLOW | deploy-X | `agent-telegram`, `telegram-voice-group`, `rho-telegram-alerts` | `channels` (§4C.7) | L4 stale version; L6 lists 13 blocked CLI commands but `channels` subcommand exists on 2026.4.15 per §4C.7; raw Telegram Bot API fallback is version-proof and works; L91-99 "14 topics" Matt-Berman default is heavy — should be "example" not "default" for base | REWRITE |
| 4 | `deploy-prompt-guide.md` | YELLOW | deploy-X | none | none | CORRECTED from ORANGE after reading L180-295: Universal Principles (L189-239) are model-agnostic and sharp (Drop Urgency Theater, Explain the Why, Show Only Desired Behavior, Remove Doubt-Based Triggers, Format Breeds Format). File already has Model-Specific Tips for Claude Opus 4.6, DeepSeek V3.x, GPT 4.x/5.x, Gemini 2.x/3.x, and "Other Models" (L247-294). Content is solid. Only issues: L4 version header shows 2026.2.26 (~8 weeks old — not years); L76/L80/L140 label Claude Opus 4.6 as "default" which should be model-agnostic; `PROMPTING-GUIDE.md` isn't a standard workspace file per §5 but that's fine as a custom add | REWRITE (light — swap "Claude Opus 4.6 default" for model-agnostic, update version header) |
| 5 | `deploy-memory.md` | YELLOW | deploy-X | none | `memory` (top-level subcommand per §4C.7) | L4 stale version; L6-7 correctly identifies memory subsystem as working; L49 `openclaw memory status --json` ✅ valid; L64 `auth-profiles.json` path ✅ matches §2; L67 correctly flags that memory keys must be in auth-profiles.json not `.env` — EXCELLENT gotcha to preserve; L88 prereq `openclaw status` — not a top-level subcommand per §4C.7 (use `openclaw health`); one of the most reliable skills in the library | REWRITE (light — just fix version header + status→health) |

## Group 4: Core data systems

| # | Skill | Score | Format | ClawHub Eqv | Bundled Eqv | Evidence | Action |
|---|-------|-------|--------|-------------|-------------|----------|--------|
| 6 | `deploy-personal-crm.md` | ORANGE | deploy-X | `personal-crm`, `heleni-personal-crm`, `crm` (§B) | `memory` (partial coverage via semantic search §4C.7) | L4 dated 2026.2.26 (~8 weeks); self-declared BLOCKED (L5); L6 lists `openclaw cron add/list`, `openclaw skills install/list`, `openclaw config get/set` as blocked — all VERIFIED AVAILABLE on 2026.4.15 per §3/§4C.4/§4C.7; underlying 20-table SQLite + `gog` + Telegram routing approach sound; scheduling can now use native `openclaw cron` | REWRITE (update BLOCKED claims; switch to native cron; evaluate `personal-crm` ClawHub skill as alternative) |
| 7 | `deploy-personal-crm-v2.md` | RED | deploy-X | (same as row 6) | same | Same file already present in `_enterprise-v2-reference/deploy-personal-crm-v2.md` verbatim (§ carve). Per user directive (2026-04-19): "reference v2 is a highly specialized version for a venture capital firm… have an extra folder… but let's not do them over anything." V2 in `skills/` duplicates the reference copy and shouldn't be a base-level default. L4 version 2026.3.27; L7 Notion-first design is engagement-specific | DELETE (from skills/; reference copy stays in `_enterprise-v2-reference/`) |
| 8 | `deploy-knowledge-base.md` | ORANGE | deploy-X | `private-knowledge-base`, `rag-search`, `hk101-living-rag`, `sulada-knowledge-base` (§B) | `memory` (partial — no URL ingestion) | L4 dated 2026.2.26; L5 self-declared BLOCKED on spurious grounds — `openclaw config get/set` ARE available (§4C.7); L9 warns about parallel system — valid concern, and §4C.5 shows `memory` is bundled (covers semantic search but not URL/YouTube/X ingestion); creates own package.json + node_modules + SQLite | REWRITE (acknowledge bundled `memory` for search; keep URL-ingestion pipeline as the differentiator; or evaluate `private-knowledge-base` ClawHub skill as replacement) |
| 9 | `deploy-google-workspace.md` | YELLOW | deploy-X | `@steipete/gog` (158k downloads, §B) | none | L4 dated 2026.2.26; L6 correctly notes no openclaw CLI deps (uses `gog` throughout); L7 flags known remote OAuth state-mismatch bug with expect workaround — good note to preserve; `@steipete/gog` IS the canonical gog skill on ClawHub — our skill is essentially the install + OAuth walkthrough for that CLI | REWRITE (light — update version header; consider REPLACE-WITH-CLAWHUB for `@steipete/gog` if the operator wants to simplify) |
| 10 | `deploy-db-backups.md` | YELLOW | deploy-X | `openclaw-backup`, `claw-backup`, `git-crypt-backup` (§B) | none | **CORRECTED 2026-04-19 per adversarial review.** L4 dated 2026.2.26; `openclaw telegram send` appears in **4 locations, not just L6**: header L6 + generated script body L216-218 + troubleshooting L597 + troubleshooting L677. Also L273 `tar -C /` is foot-gun on SELinux/Gatekeeper; L313-318 retention logic broken if stray files match glob; L324-330 uses quoted-heredoc placeholder-hand-edit anti-pattern. Core GPG+gog approach still sound. | **REWRITE (medium)**: sweep ALL 4 `openclaw telegram send` occurrences (not just L6); fix L313 retention glob; split L324 heredoc write from placeholder substitution |
| 11 | `deploy-social-tracking.md` | ORANGE | deploy-X | individual integrations only (no multi-platform aggregator) | none | L4 dated 2026.2.26; L5 BLOCKED; L6 lists `openclaw config set`, `openclaw cron add/list/remove` as blocked — ALL VERIFIED AVAILABLE on 2026.4.15 per §3/§4C.7; heavy (Python deps + YouTube/IG/X/TikTok auth); per §B ClawHub has `tiktok-uploader`, `publora-tiktok`, `upload-post` (cross-platform) but no direct "daily-analytics-snapshot" skill | REWRITE (BLOCKED claims all wrong; use native `openclaw cron` + workspace `.env` config) |

## Group 5: Intelligence systems

| # | Skill | Score | Format | ClawHub Eqv | Bundled Eqv | Evidence | Action |
|---|-------|-------|--------|-------------|-------------|----------|--------|
| 12 | `deploy-advisory-council.md` | ORANGE | deploy-X | `council-of-the-wise` (§B) | none | L4 dated 2026.2.26; L5 BLOCKED; L6 lists 7 blocked commands (cron + config + onboard) — all exist on 2026.4.15; L44 hardcodes `ANTHROPIC_API_KEY` — violates §BB Codex-first; 8 parallel AI personas = expensive, heartbeat override broken (§K) so can't cheaply schedule; architecturally interesting | REWRITE (unblock commands; swap Anthropic→Codex per §BB; document cost) |
| 13 | `deploy-security-council.md` | ORANGE (superseded) | deploy-X | `security-reviewer`, `pr-reviewer` (diff scope, §B) | none | Superseded by v2 (row 14). BUT L10 "`openclaw agent -m "PROMPT" --json --timeout 120` WORKS" note (Gregory Ringler 2026-03-26) is valuable. L21-38 documents critical 64-hour outage (a mentor 2026-03-08) from migrating `.env` to macOS Keychain under LaunchAgent — INCIDENT LEARNINGS MUST BE PRESERVED | DELETE — but ONLY after L21-38 Keychain-incompatibility warning is merged into v2's persona prompts (or into `_authoring/` anti-patterns) |
| 14 | `deploy-security-council-v2.md` | YELLOW | deploy-X | same as row 13 | none | L4 "2026.2.26+"; L5 READY; L6 zero CLI deps; standalone JS + launchd + Anthropic API direct — solid architecture; L14 calls Anthropic directly which violates §BB Codex-first default; macOS-only (launchd); 4 security personas ≠ 8 so cheaper than advisory-council; NEEDS the Keychain-incompatibility warning from row 13 embedded | REWRITE (light — swap Anthropic→Codex default; merge the L21-38 incident warning from row 13 so we don't lose that lesson) |
| 15 | `deploy-platform-health.md` | ORANGE | deploy-X | none (unique 9-area health analysis) | `doctor` subcommand (§4C.7) covers basic health | L4 dated 2026.2.26; L5 BLOCKED; L6 4 blocked commands (cron + config) — all exist on 2026.4.15; 9 daily health areas via codebase AI analysis; overlaps with built-in `openclaw doctor` for basic health but goes deeper | REWRITE (unblock commands; evaluate overlap with bundled `doctor` — keep only the delta) |
| 16 | `deploy-mission-control.md` | YELLOW | deploy-X | none | `dashboard` subcommand (§4C.7) "Open the Control UI with your current token" — bundled alternative | L1 wraps `robsannaa/openclaw-mission-control` (3rd-party GitHub project; our §P research saw `abhi1693/openclaw-mission-control` — may be two different projects or naming drift); §8 notes 2026.4.18 shipped Control UI overhaul with presets/quick-create; bundled `openclaw dashboard` may obsolete this entirely | REWRITE (verify robsannaa repo exists/current; compare feature matrix against bundled Control UI; REPLACE-WITH-BUNDLED if equivalent) |

## Group 6: Productivity / daily flow

| # | Skill | Score | Format | ClawHub Eqv | Bundled Eqv | Evidence | Action |
|---|-------|-------|--------|-------------|-------------|----------|--------|
| 17 | `deploy-daily-briefing.md` | ORANGE | deploy-X | `morning-daily-briefing`, `ai-daily-briefing` (§B) | none | L4 dated 2026.2.26; L5 BLOCKED; L6 4 blocked cmds (cron + config briefing.*) — cron is available on 2026.4.15 per §3; config namespaces are a separate question; core 7am Telegram-briefing concept solid | REWRITE (unblock cron per §3; drop `briefing.*` config namespace — put briefing config in workspace files per §4) |
| 18 | `deploy-daily-briefing-v2.md` | RED | deploy-X | same as row 17 | none | Duplicate of `_enterprise-v2-reference/deploy-daily-briefing-v2.md`. L11 explicitly "for a VC Managing Director" — engagement-specific. Uses Claude API direct (violates §BB Codex-first). Per user directive 2026-04-19 "don't prefer reference v2" | DELETE (from skills/; reference copy preserved in `_enterprise-v2-reference/`) |
| 19 | `deploy-urgent-email.md` | ORANGE | deploy-X | `email-triage`, `expanso-email-triage`, `email-triage-pro` (§B) | none | L4 dated 2026.2.26; L5 BLOCKED; L6 4 blocked commands (config + cron) — cron is available on 2026.4.15; core AI classification + feedback loop sound | REWRITE (unblock cron; put config in workspace files) |
| 20 | `deploy-urgent-email-v2.md` | RED | deploy-X | same as row 19 | none | Duplicate of `_enterprise-v2-reference/deploy-urgent-email-v2.md`. L13 explicitly "for {{PRINCIPAL_NAME}} (Managing Director, {{COMPANY_NAME}})". Claude Haiku hardcoded (§BB violation). Himalaya + Notion + SQLite feedback all engagement-specific | DELETE (reference copy preserved) |
| 21 | `deploy-action-items.md` | YELLOW | deploy-X | none directly (`email-triage-pro` covers some §B) | none | L4 "2026.3.22+", READY, no blocked commands; single-person approval flow (diff vs {{FAMILY_NAME}} ref shows {{FAMILY_NAME}} has multi-person routing, ours is simpler); Claude hardcoded — §BB violation | REWRITE (swap Claude → Codex per §BB) |
| 22 | `deploy-meeting-prep.md` | **ORANGE (P1 DEADLINE)** | deploy-X | `meeting-prep`, `ai-meeting-prep`, `meeting-prep-agent` (§B) | none | **CORRECTED 2026-04-19 per adversarial review.** L121 hardcodes `claude-sonnet-4-20250514` — this snapshot **RETIRES 2026-06-15** per baseline §X. Every client install using this skill breaks in <2 months. L44 leaks {{COMPANY_NAME}} email; L47 example `${TELEGRAM_CHAT_ID}={{PRINCIPAL_CHAT_ID}}` leaks {{PRINCIPAL_NAME}}'s Telegram user ID. Raw API approach otherwise sound. | **REWRITE (P1 — pre-June 15)**: replace `claude-sonnet-4-20250514` → `claude-sonnet-4-6` (stable alias) or defer to agent default per §BB; scrub client identifiers |
| 23 | `deploy-transcript-pipeline.md` | YELLOW | deploy-X | `video-transcript-downloader`, `youtube-transcript` (partial — source step only) | none | L4 "2026.3.22+", READY; standalone scripts + SQLite tracking + launchd + raw APIs (Google Drive + Fathom + Notion + Anthropic); Anthropic hardcoded — §BB violation | REWRITE (Codex swap) |
| 24 | `deploy-fathom-pipeline.md` | ORANGE | deploy-X | `fathom-api`, `fathom-meetings`, `fathom` (§B) | none | L4 dated 2026.2.26, BLOCKED; L6 4 blocked cmds (cron available on 2026.4.15 per §3; `config set-env` unknown); Fathom-specific ({{FAMILY_NAME}} uses Gemini alternative per project memory); community `fathom-api` skill exists on ClawHub | REWRITE or REPLACE-WITH-CLAWHUB (if `@-/fathom-api` is maintained) |

## Group 7: Integrations

| # | Skill | Score | Format | ClawHub Eqv | Bundled Eqv | Evidence | Action |
|---|-------|-------|--------|-------------|-------------|----------|--------|
| 25 | `deploy-notion-workspace.md` | YELLOW | deploy-X | `@steipete/notion`, `notion-skill`, `notion-api-skill` (§B) | none | L4 "2026.3.22", READY, no blocked; skills/ is a subset of {{FAMILY_NAME}} ref (ref adds 94-line "Approval Gates & Oversight Agent" block — engagement-specific, correctly omitted here); raw Notion API approach is solid | REWRITE (light — check for Claude refs) |
| 26 | `deploy-linear-integration.md` | YELLOW | deploy-X | `linear-api` (managed-OAuth), `linear-skill`, `linear-autopilot` (§B) | none | L4 "2026.3.22+", READY, no blocked; ~identical to {{FAMILY_NAME}} ref; raw Linear GraphQL + launchd + SQLite cache approach is sound | REWRITE (light) or REPLACE-WITH-CLAWHUB for `linear-api` |
| 27 | `deploy-github-cicd.md` | **ORANGE (engagement-saturated)** | deploy-X | `github-code-review-cicd`, `@steipete/github` (160k dl §B) | none | **CORRECTED 2026-04-19 via client-identifier sweep.** L13 When-to-use says "for {{COMPANY_NAME}} projects"; L45 default `${DEV_EMAIL}=dev@<INTERNAL_DOMAIN>`; L55 `${CUSTOM_DOMAIN}=app.<INTERNAL_DOMAIN>`; L248 hardcodes `"{{AGENT_NAME}} ({{COMPANY_NAME}} Dev)"` as git user name. Beyond "examples" — structurally engagement-flavored. Technical content is fine (git/ssh/gh all stable). | **NEEDS OPERATOR DECISION**: DELETE (use `_enterprise-v2-reference/` as the source of truth for engagement-specific CI/CD) OR full REWRITE (strip all {{COMPANY_NAME}} refs, replace with generic dev/domain placeholders) OR REPLACE-WITH-CLAWHUB for `@steipete/github` |
| 28 | `deploy-himalaya-email.md` | **ORANGE (engagement-saturated, macOS-only)** | deploy-X | `gws-gmail-send`, `@steipete/gog` (different auth model) | none | **CORRECTED 2026-04-19 via adversarial review + client-identifier sweep.** L11 Purpose says "for the {{AGENT_NAME}} AI executive assistant" (engagement-specific); L17 "IMAP/SMTP access to `edge@{{COMPANY_DOMAIN}}`"; L22 "auto-forward detection ({{PRINCIPAL_NAME}} forwards...)"; L40 example `${EMAIL_ADDRESS}=edge@{{COMPANY_DOMAIN}}`; L464 regex hardcodes `/principal@{{COMPANY_SLUG}}\.com/i`; L878 array contains `'{{PRINCIPAL_EMAIL}}'`. L62 declares macOS-only (undocumented in compat block). L89 says Node 18+ (below OpenClaw's 22.14+ minimum per §1). Himalaya + App Passwords approach technically sound, but content is fully engagement-flavored. | **NEEDS OPERATOR DECISION**: DELETE (reference-snapshot copy preserved) OR heavy REWRITE (scrub {{FAMILY_NAME}} structural refs, add explicit macOS-only + Node 22.14+ labels, generalize) — NOT a light fix |
| 29 | `deploy-asana.md` | ORANGE | deploy-X | `asana-api` (managed-OAuth), `asana-pat`, `asana-agent-skill` (§B) | none | L4 dated 2026.2.26, BLOCKED; L6 2 blocked cron cmds — BOTH available on 2026.4.15 per §3; Asana API logic should work | REWRITE (unblock cron per §3) or REPLACE-WITH-CLAWHUB for `asana-api` |
| 30 | `deploy-newsletter-crm.md` | ORANGE | deploy-X | `beehiiv-integration`, `beehiiv` (managed-OAuth), `newsletter-digest` (§B) | none | L4 dated 2026.2.26, BLOCKED; L6 4 blocked cmds — 3 cron (available on 2026.4.15 per §3) + `openclaw status` (doesn't exist — use `openclaw health` per §4C.7); core Beehiiv/HubSpot API logic should work | REWRITE (unblock cron; `status`→`health`) or REPLACE-WITH-CLAWHUB for `beehiiv` |
| 31 | `deploy-telegram-forum.md` | YELLOW | deploy-X | `agent-telegram`, `telegram-voice-group`, `rho-telegram-alerts` (§B) | `channels` (§4C.7) | L4 dated 2026.2.26, PARTIALLY WORKING; L7 uses raw Bot API — version-proof; overlap with bundled `channels` subcommand | REWRITE (light — verify vs bundled `channels`) |

## Group 8: Content & creative

| # | Skill | Score | Format | ClawHub Eqv | Bundled Eqv | Evidence | Action |
|---|-------|-------|--------|-------------|-------------|----------|--------|
| 32 | `deploy-humanizer.md` | YELLOW | deploy-X | **`@biostartechnology/humanizer` (92.8k dl §B)**, `ai-humanizer`, `humanizer-enhanced` | none | L4 2026.2.26, NEEDS AUDIT; L6 `openclaw chat` may not exist (§4C.7 has `agent` not `chat`); workspace-file-based skill; ClawHub has a very popular alternative | **REPLACE-WITH-CLAWHUB** for `@biostartechnology/humanizer` (community-maintained, battle-tested) |
| 33 | `deploy-image-gen.md` | YELLOW | deploy-X | **`@steipete/nano-banana-pro` (88.1k dl, Gemini 3 Pro §B)**, `best-image-generation`, `cheapest-image-generation` | none | L4 2026.2.26, NEEDS AUDIT; no openclaw CLI refs; Gemini-based; ClawHub `nano-banana-pro` IS the Gemini 3 Pro image skill | **REPLACE-WITH-CLAWHUB** for `@steipete/nano-banana-pro` |
| 34 | `deploy-video-gen.md` | YELLOW | deploy-X | `video-generation-minimax`, `giggle-generation-video` (§B) | none | L4 2026.2.26, NEEDS AUDIT; no openclaw CLI refs; Gemini Veo 3 | REWRITE (light — version header; verify Veo 3 API still current) |
| 35 | `deploy-video-analysis.md` | YELLOW | deploy-X | none directly | none | L4 2026.2.26, NEEDS AUDIT; no openclaw CLI refs; Gemini video analysis | REWRITE (light — version header) |
| 36 | `deploy-video-pipeline.md` | ORANGE | deploy-X | partial (Asana, Slack integrations on ClawHub) | none | L4 2026.2.26, BLOCKED; L6 6 blocked cmds: `openclaw asana` + `openclaw slack status` genuinely never existed (wrong names); `openclaw skills install` VALID per §4C.4 (skill→skills typo); `openclaw config get/set` VALID per §4C.7 — half the "blocked" list is skill-author error | REWRITE (heavy — fix blocked-cmd mislabels, rewrite Asana/Slack integrations to use direct APIs) |
| 37 | `deploy-video-research.md` | YELLOW | deploy-X | none directly | none | L4 2026.2.26, READY, no blocked | REWRITE (light — version header) |
| 38 | `deploy-content-pipeline.md` | YELLOW | deploy-X | none directly | none | L4 2026.2.26, READY, no blocked | REWRITE (light — version header) |
| 39 | `deploy-youtube-posting.md` | YELLOW | deploy-X | `upload-post` (cross-platform §B), `postfast` | none | L4 2026.2.26, READY | REWRITE (light) or REPLACE-WITH-CLAWHUB for `upload-post` if cross-platform value matters |
| 40 | `deploy-tiktok-account-setup.md` | YELLOW | deploy-X | `tiktok-uploader`, `publora-tiktok`, `tiktok-growth` (§B) | none | L4 2026.2.26, READY; TikTok-specific workflow | REWRITE (light) or REPLACE-WITH-CLAWHUB |
| 41 | `deploy-tiktok-compliance.md` | YELLOW | deploy-X | partial (compliance not specifically on ClawHub) | none | L4 2026.2.26, READY; TikTok compliance gate (policy check before post) — potentially unique value | REWRITE (light) — keep as differentiator |
| 42 | `deploy-tiktok-posting.md` | YELLOW | deploy-X | `tiktok-uploader`, `publora-tiktok`, `upload-post` (§B) | none | L4 2026.2.26, READY | REPLACE-WITH-CLAWHUB for `upload-post` (covers TikTok + 9 others) preferred |
| 43 | `deploy-tiktok-warmup.md` | YELLOW | deploy-X | `tiktok-growth` (§B) | none | L4 2026.2.26, READY; Account warmup is unusual | REWRITE (light) — potentially unique |

## Group 9: Operations

| # | Skill | Score | Format | ClawHub Eqv | Bundled Eqv | Evidence | Action |
|---|-------|-------|--------|-------------|-------------|----------|--------|
| 44 | `deploy-git-autosync.md` | RED | deploy-X | `git-essentials`, `git-workflows` (neither does hourly autocommit, §B) | none | Superseded by v2 (row 45). L4 2026.2.26; L6 7 blocked cmds — cron/config available on 2026.4.15 per §3/§4C.7; `status`→`health`; `daemon` is a legacy alias per §4C.7 (not blocked); v2 is the generic replacement | DELETE (superseded by v2 which is generic) |
| 45 | `deploy-git-autosync-v2.md` | YELLOW | deploy-X | (no exact ClawHub equivalent for hourly autocommit §B) | none | L4 no version (READY); standalone scripts + launchd; identical to {{FAMILY_NAME}} ref but content is generic (git/ssh/launchd/bash/curl); one of 2 unique value-adds (no ClawHub equivalent for hourly autocommit per §B) | REWRITE (light — could simplify to use `openclaw cron` now available on 2026.4.15 instead of launchd) |
| 46 | `deploy-health-monitoring.md` | **RED (architecture)** | deploy-X | none | `doctor` (basic health §4C.7) | **CORRECTED 2026-04-19 per adversarial review.** L4 2026.2.26; compat header unblocked in 2026-04-19 rewrite (commands exist on 2026.4.15). **BUT architectural issue: skill fires 3 separate cron jobs (30-min, weekly, nightly) — directly violates baseline §O community anti-pattern "don't fan out N cron jobs — collapse into heartbeat."** Bundled `openclaw doctor` + heartbeat API cover most of this natively. Not a version problem; a design problem. | **REDESIGN to single heartbeat** per §L (cheap-checks-first, model-on-alert) — or DELETE if `openclaw doctor` + bundled heartbeat suffice. Compat-header rewrite from earlier today stays, but body needs structural redesign. |
| 47 | `deploy-model-tracking.md` | ORANGE | deploy-X | none (no model-cost dashboard on ClawHub per §B) | none (no bundled model tracker) | L4 2026.2.26, BLOCKED; L6 4 cmds — `openclaw skill install` typo for `skills install` (valid §4C.4); `openclaw config get/set` valid (§4C.7); `openclaw doctor` valid — ALL 4 claims wrong; genuine value-add (no community equivalent) | REWRITE (fix blocked-cmd mislabels; unique skill worth keeping) |
| 48 | `deploy-earnings.md` | ORANGE | deploy-X | `earnings-reader`, `earnings-calendar`, `stock-earnings-review` (§B) | none | L4 2026.2.26, BLOCKED; L6 5 cmds — cron/config/show available; `status`→`health`; stock-market earnings (not personal revenue); ClawHub has 3 community alternatives | REWRITE or REPLACE-WITH-CLAWHUB for `earnings-calendar` |
| 49 | `deploy-food-journal.md` | ORANGE | deploy-X | `food` (partial only — no scaled food journal §B) | none | L4 2026.2.26, BLOCKED; L6 5 cmds — cron/config/show available; `status`→`health`; 3x daily Telegram meal prompts + weekly correlation analysis; largely unique value-add | REWRITE (unblock cmds; unique skill) |
| 50 | `oauth-reauth.md` | YELLOW | deploy-X | none (utility skill) | `models auth login` bundled (§4C.4) covers basic reauth | L4 "2026.3.2+", WORKING; L6 notes `openclaw models auth login` requires TTY — documents `screen` workaround over device exec; telegram-guided user interaction | REWRITE (light — verify against 2026.4.15; `models auth` is §4C.4 bundled, may overlap) |
| 51 | `token-optimization.md` | YELLOW | deploy-X | `session-cost-tracker` (partial, §B) | `models fallbacks` + `models set` (§4C.4) | No Compatibility block; about cost-efficient model defaults, aliases, fallback chains, prompt caching, local heartbeat routing — ALL directly relevant to Flyn's cheap-first design; overlaps with bundled `openclaw models` subcommands | REWRITE (heavy — fold in Codex-default per §BB; use bundled `openclaw models set`/`aliases`/`fallbacks` vs manual config editing) |
| 52 | `diagnose-cron.md` | **GREEN** | deploy-X | none | `cron status`, `cron runs` (file already uses these) | **CORRECTED 2026-04-19 per adversarial review.** File ALREADY uses `openclaw cron status` (L131), `openclaw cron runs --id <jobId> --limit 10` (L128), and `openclaw cron add` with modern flags (L116-122, `--cron`, `--tz`, `--session isolated`, `--announce --channel telegram --to`). Written against 2026.3.x+ (L220 says "OpenClaw version 2026.3.13"). **The most up-to-date skill in the library.** L13-19 Known Issues Registry is library-wide gold (7 documented cron bugs). Only polish: L220 version bump, L213-221 Adaptation Points hardcodes `<TELEGRAM_GROUP_ID> (Rel HQ)` and topic `156 (Sprint)` — placeholder-ize | **KEEP — 15-min polish only** (bump L220 to 2026.4.15; replace Rel chat IDs with `<CLIENT_CHAT_ID>`/`<CLIENT_TOPIC_ID>` placeholders) |

---

## Skipped (intentionally)

- `skills/_archive/*` — already archived in source; not in scope
- `skills/_enterprise-v2-reference/*` — reference only, not a deployment target

---

## Per-Skill Detail Blocks

### #57 `_authoring/_deploy-common.md` — ORANGE

**Lines reviewed:** full file, 649 lines
**Evidence:**
- **Line 9 / "Philosophy" section:** "An AI operator (Claude Opus 4.6) doesn't need exact commands copy-pasted." — hardcodes Claude Opus 4.6 as the operator. Violates `feedback_codex_default_models.md` (baseline §BB).
- **Lines 219-221 / "API Key Validation":** Hardcodes Anthropic-only key prefixes (`sk-ant-api03-`, `sk-ant-oat01-`) as the validation rule. Should accommodate OpenAI/Codex keys + OAuth flows (Codex is primary per baseline §Y).
- **Lines 582-586 / "OpenClaw General" gotchas table:** claims `openclaw restart`, `openclaw cron`, `openclaw telegram`, `openclaw config set messaging.*` all "do not exist in 2026.2.26". **LIVE 4C PROBE (§4C.3, §4C.7) PROVES**: on 2026.4.15, `openclaw cron` IS a first-class subcommand (add/disable/edit/enable/list/rm/run/runs/status). `channels` is a subcommand (replaces old `telegram`). `messaging` namespace — need verification. These lines will mislead anyone reading.
- **Lines 78-147 / "Phase 0.5 OpenClaw Compatibility Check":** EXCELLENT and evergreen. Teaches: record version, list skills, dump config, verify each subcommand — this stays.
- **Lines 150-180 / "File Transfer Standard":** base64 encoding rule is correct and important. Stays.
- **Lines 250-366 / "Telegram Client Interaction":** 116 lines of OAC-specific Telegram polling, webhook trap/restore, 2-phase retry. This is Rel/OAC domain, not generic openclaw-base. Should move to a separate `_oac-telegram-client.md` if preserved at all.
- **Lines 546-591 / "Platform Gotchas":** very good — macOS crontab hangs, launchd absolute paths, Windows BOM, Linux loginctl linger, etc. Most still valid. The "OpenClaw General" subtable (L580-591) needs full rewrite against 2026.4.15.
- **Entire file is OAC-flavored** (Session API, Device exec, Bearer tokens, client profiles) — openclaw-base is meant to be generic OpenClaw deployment, not OAC-operator deployment.

**Recommendation:** Major rewrite. Split into two docs:
1. `_deploy-common.md` (generic, ≤250 lines): Philosophy, 3-level abstraction model, Phase 0.5 compatibility check, file transfer base64, variables table, autonomy modes, idempotency patterns, error handling. Codex-default. No OAC-specific content.
2. `_oac-operator-protocol.md` (OAC-only, moved elsewhere eventually): Device exec API, session commands API, bearer tokens, Telegram polling for operator interactions, client profiles.

Platform Gotchas table should be rewritten against OpenClaw 2026.4.15 — remove the "doesn't exist" falsehoods, add new gotchas from baseline §M (heartbeat model override broken, Matrix 2026.2.17, Ollama streaming).

**ClawHub alternative considered:** n/a (meta-doc, not a skill to install)
**Bundled alternative considered:** n/a

---

### #58 `_authoring/_skill-authoring-guide.md` — YELLOW

**Lines reviewed:** full file, 434 lines
**Evidence:**
- **Line 9 / "Philosophy":** "The operator (Claude Opus 4.6) is highly capable" — hardcodes Claude Opus 4.6. Violates §BB.
- **Line 220 / Full Example step 2.3:** example writes `"model": "claude-sonnet-4-20250514"` to a config. Per baseline §X, `claude-sonnet-4-20250514` deprecated 2026-04-14, **retires 2026-06-15**. Skill written against this example will break in 2 months.
- **Lines 232-321 / Anti-Patterns section:** 10 concrete anti-patterns (fictional CLI, hardcoded paths, `echo '\n'`, quoted heredocs with variables, `echo -e`, `for file in $(cmd)`, macOS-only package managers, hardcoded topic IDs, orphaned scripts, `wc -c` on macOS). All correct, evergreen. Keep verbatim.
- **Lines 336-352 / Cross-Platform Reference Table:** solid. Keep.
- **Lines 24-30 / "3-Level Abstraction Model":** Level 1/2/3 framework (exact syntax / intent + reference / intent + knowledge) is a good operator-runbook idea, but doesn't translate to canonical SKILL.md (which is agent-loaded, not operator-executed). Clarify that this model applies to runbook-style skills specifically.
- **Lines 356-434 / "New Skill Template":** uses `# Deploy [System Name]` H1 followed by prose sections (`## Purpose`, `## How Commands Are Sent`, etc.). This is OUR format. Canonical OpenClaw SKILL.md (per baseline §6) uses YAML frontmatter: `name: <snake_case>`, `description: <one line>` — then markdown body. These coexist but the guide doesn't mention SKILL.md format at all.

**Recommendation:** YELLOW-level rewrite:
1. Global s/Claude Opus 4.6/OpenAI Codex GPT-5.4/ in voice-of-operator phrasing.
2. Replace `claude-sonnet-4-20250514` in L220 example with `openai-codex/gpt-5.4` and note that hardcoded model IDs in skills are themselves an anti-pattern (skills should defer to `agents.defaults.model`).
3. Add a new section "Two Skill Formats": explain OAC-runbook deploy-X.md (this guide) vs canonical SKILL.md (for agent-loadable skills, ClawHub-publishable). Link both.
4. Keep Anti-Patterns + Cross-Platform tables as-is — they're the best part of the doc.

**ClawHub alternative considered:** n/a
**Bundled alternative considered:** n/a

---

### #53 `install/openclaw-install.md` — ORANGE

**Lines reviewed:** full file, 635 lines
**Evidence:**
- **L159-430 / Phase 1 "Persist Remote Access":** entirely OAC-specific. Enrollment, device ID, bearer tokens, LaunchAgent for `~/.openclaw-remote/`, `api.openclawinstall.net` endpoints. This is Rel/OAC territory, not generic OpenClaw install.
- **L433-527 / Phase 2 "Install OpenClaw":** mostly correct per baseline §1.
  - L443 `curl -fsSL https://openclaw.ai/install.sh | bash` ✅ matches §1
  - L448 Windows `irm | iex` ✅ matches §1
  - L453 `npm install -g openclaw@latest` ✅ matches §1
  - L483 `openclaw onboard --install-daemon --non-interactive` ✅ matches §1
- **L503 `openclaw status`:** per live 4C probe (§4C.7), top-level `status` is NOT a subcommand. Current 2026.4.15 has `openclaw health` and `openclaw gateway status`. This command will fail.
- **L515 `openclaw doctor`:** ✅ valid per §4C.7
- **L634 "Related:" line:** references `skills/archive/setup-windows.md` etc. — those archive paths still exist but would be moved; reference is stale.
- **No mention of the canonical `SKILL.md` format per §6** — this skill only teaches our `deploy-X.md` runbook format.

**Recommendation:** REWRITE as a clean generic install runbook:
1. DELETE entire Phase 1 (OAC-specific). This content moves to OAC's own install runbook.
2. KEEP Phase 2 as-is but:
   - Fix L503: `openclaw status` → `openclaw health` (or `openclaw gateway status`).
   - Fix L634: remove archived-path references; point to the single canonical install guide.
   - Add a preamble noting: "This covers local OpenClaw install. For persistent remote-access agent enrollment, see OAC-specific runbooks."
3. Length target: ~150 lines (vs current 635).

**ClawHub alternative considered:** n/a (install is fundamental)
**Bundled alternative considered:** n/a

---

### #54, #55, #56 `install/setup-{macos,windows,ubuntu-gce}.md` — RED (DELETED 2026-04-19)

**Lines reviewed:** full files (357, ~400, ~300 lines respectively)
**Evidence:**
- All three files document **OAC agent enrollment**, not OpenClaw install. Session-mode one-liner → verification code → device enrollment API → persistent agent via launchd/schtasks/systemd for `~/.openagent/agent/` or `C:\OpenClaw\agent.js`.
- OpenClaw installation on all three platforms is trivial per baseline §1: one command (`curl -fsSL https://openclaw.ai/install.sh | bash` or `iwr | iex`). These 1,000+ lines teach a different system.
- `setup-ubuntu-gce.md:43-44` installs Node 20 via NodeSource — **below OpenClaw's minimum** of Node 22.14+ (baseline §1). Actively wrong for OpenClaw install.
- `setup-macos.md` and `setup-windows.md` describe `/connect/TOKEN` one-liners that download `macos-arm64.zip` / PowerShell bundles containing `node` + `agent.js` — this is the OAC agent bundle, distinct from OpenClaw binaries.

**Action executed:** Files deleted from `install/`. Content preserved in `rel-hq/repos/rel-agent-ARCHIVED-2026-04-05/skills/setup-*.md` for future OAC repo migration.

**Replacement plan:**
- Single cross-platform `install/openclaw-install.md` (rewrite per #53) covers macOS/Linux/Windows via the official install scripts. Platform-specific quirks (macOS Gatekeeper, Windows execution policy, Ubuntu Node version) go in a small Troubleshooting section.

**ClawHub alternative considered:** n/a (install is fundamental)
**Bundled alternative considered:** n/a

---

### #59 `_authoring/_skill-review.md` — ORANGE

**Lines reviewed:** full file, 192 lines
**Evidence:**
- **Line 106 / "Reviewer 3: AI Operability":** "Can an AI agent (Opus 4.6) execute this skill through the session commands API?" — hardcodes Opus 4.6 AND assumes OAC session API as the delivery mechanism. Neither is right for openclaw-base.
- **Lines 183-188 / "Running the Review":** "Launch 3 agents in parallel: Agent 1 (Format), Agent 2 (Accuracy), Agent 3 (Operability)." **Directly violates `feedback_no_subagent_audits.md`** — the operator's explicit rule: per-item reviews must be done by Claude directly, not delegated to subagents, because subagents fake-confirm without actually reading.
- **Lines 23-34 / "Required sections present" checklist:** 12 mandatory sections including `## How Commands Are Sent`, `## Variables` (with Source/Example columns), `## Adaptation Points` — these are OAC-operator-runbook conventions. Canonical SKILL.md (§6) has completely different required shape: YAML frontmatter with `name` + `description`, then body with Quick Start → Workflow → Commands → Example Session → Requirements → Configuration. This reviewer would mark every ClawHub skill as non-compliant.
- **Good parts:**
  - 3-reviewer lens (Format / Technical Accuracy / AI Operability) — solid framework.
  - Severity definitions (L172-179) are clear and useful.
  - Shell-quoting checks (L69-77) are accurate and evergreen.
- **Structural issue:** review criteria are all bolted to our `deploy-X.md` format. Doesn't know what to do with a canonical SKILL.md.

**Recommendation:** ORANGE-level rewrite:
1. Remove "Launch 3 agents in parallel" section entirely. Replace with "Claude reviews each skill directly, one at a time, filling in per-skill findings inline."
2. Fork into two reviewers: `_skill-review-runbook.md` (for our deploy-X.md format) and `_skill-review-skillmd.md` (for canonical SKILL.md skills).
3. Global s/Opus 4.6/Codex GPT-5.4/ + update any fictional-CLI callouts.
4. Keep the 3-lens concept and severity definitions.

**ClawHub alternative considered:** `skill-vetter` (https://clawhub.ai/@spclaudehome/skill-vetter) is popular (214k downloads) but it's a PRE-INSTALL SECURITY scanner, not a format/quality reviewer — different purpose. No direct equivalent for "quality audit of a deploy runbook."
**Bundled alternative considered:** n/a

---
