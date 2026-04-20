# ClawHub recommendations — index

This directory holds one file per ClawHub skill that `openclaw-base` recommends as a preferred alternative (or adjunct) to our own `deploy-*.md` skills. Each file has YAML frontmatter with the fields an audit agent needs to verify the skill (ClawHub URL, author, license, version, last-checked date, maintenance status).

These are **recommendations**, not installs. They are community skills we've evaluated as higher-leverage than maintaining equivalent `deploy-*.md` logic ourselves, OR as net-new capabilities that complement our stack.

**Parent baseline:** [`../../audit/_baseline.md`](../../audit/_baseline.md) §B — master list of ClawHub skills overlapping our themes.
**Mirror pattern:** [`../memory-options/`](../memory-options/) — same frontmatter + index shape, applied to memory-architecture options.
**Template for adding new recommendations:** [`_template.md`](./_template.md)

## Index

| File | Name | Category | Author | Status | Last checked |
|------|------|----------|--------|--------|--------------|
| [`gog.md`](./gog.md) | gog (Google Workspace CLI) | integrations | @steipete | active | 2026-04-20 |
| [`notion.md`](./notion.md) | notion | integrations | @steipete | active | 2026-04-20 |
| [`github.md`](./github.md) | github | integrations | @steipete | active | 2026-04-20 |
| [`humanizer.md`](./humanizer.md) | humanizer | content-creation | @biostartechnology | active | 2026-04-20 |
| [`nano-banana-pro.md`](./nano-banana-pro.md) | nano-banana-pro | content-creation | @steipete | active | 2026-04-20 |
| [`linear-api.md`](./linear-api.md) | linear-api | integrations | unverified | unverified | 2026-04-20 |
| [`asana-api.md`](./asana-api.md) | asana-api | integrations | unverified | unverified | 2026-04-20 |
| [`beehiiv.md`](./beehiiv.md) | beehiiv | integrations | unverified | unverified | 2026-04-20 |
| [`upload-post.md`](./upload-post.md) | upload-post | content-creation | unverified | unverified | 2026-04-20 |
| [`personal-crm.md`](./personal-crm.md) | personal-crm | crm | unverified | unverified | 2026-04-20 |
| [`email-triage-pro.md`](./email-triage-pro.md) | email-triage-pro | productivity | unverified | unverified | 2026-04-20 |
| [`morning-daily-briefing.md`](./morning-daily-briefing.md) | morning-daily-briefing | productivity | unverified | unverified | 2026-04-20 |
| [`ai-meeting-prep.md`](./ai-meeting-prep.md) | ai-meeting-prep | productivity | unverified | unverified | 2026-04-20 |
| [`private-knowledge-base.md`](./private-knowledge-base.md) | private-knowledge-base | research | unverified | unverified | 2026-04-20 |
| [`earnings-calendar.md`](./earnings-calendar.md) | earnings-calendar | research | unverified | unverified | 2026-04-20 |
| [`skill-vetter.md`](./skill-vetter.md) | skill-vetter | security | @spclaudehome | unverified | 2026-04-20 |

## Categories

- **integrations** — external-service wrappers (Google Workspace, Notion, GitHub, Linear, Asana, Beehiiv)
- **productivity** — briefings, triage, meeting prep
- **content-creation** — humanizer, image gen, cross-platform posting
- **crm** — personal / relationship tracking
- **research** — knowledge-base + external-data skills (earnings, RAG)
- **security** — skill vetting and supply-chain defense

## Cross-references to our deploy-* skills

Each recommendation file declares the deploy-* skill it maps to via the `recommends_over:` frontmatter field. Quick map:

| Our skill | Recommended ClawHub alternative |
|-----------|-------------------------------|
| `deploy-google-workspace.md` | `gog.md` |
| `deploy-humanizer.md` | `humanizer.md` |
| `deploy-image-gen.md` | `nano-banana-pro.md` |
| `deploy-asana.md` | `asana-api.md` |
| `deploy-newsletter-crm.md` | `beehiiv.md` (Beehiiv half) |
| `deploy-tiktok-posting.md` + `deploy-youtube-posting.md` | `upload-post.md` |
| `deploy-personal-crm.md` | `personal-crm.md` (evaluate first) |
| `deploy-urgent-email.md` | `email-triage-pro.md` |
| `deploy-daily-briefing.md` | `morning-daily-briefing.md` |
| `deploy-knowledge-base.md` | `private-knowledge-base.md` |
| `deploy-earnings.md` | `earnings-calendar.md` (stock-market only) |
| (none — was deleted engagement-specific) | `notion.md`, `github.md`, `ai-meeting-prep.md` |
| (no deploy-* skill) | `linear-api.md` |
| (general defense) | `skill-vetter.md` |

## How to add a new recommendation

1. Copy [`_template.md`](./_template.md) → `<slug>.md`
2. Fill in the frontmatter YAML fields. If you can't verify a value (download count, license, author handle), mark it `unverified` or `<VERIFY>` — **do not fabricate**.
3. Write the body: What it is / Why / When to pick / When not to pick / Install / Configure / Gotchas / Citations.
4. Add a row to the Index table in this README.
5. Add a row to the "Cross-references to our deploy-* skills" map if it maps to one.
6. (Optional) Add a `ClawHub alternative` line to the mapped `deploy-*.md` compat header so it cross-links back.

## For audit / update-checker agents

Every recommendation file has YAML frontmatter at the top with these machine-parseable fields:

- `name`, `clawhub_slug`, `clawhub_url`, `category`, `author`, `license`
- `downloads_last_checked`, `version_last_verified`, `last_checked` (bump when you re-check)
- `maintenance_status` (`active` / `dormant` / `deprecated` / `unverified`)
- `recommends_over` (the deploy-*.md it replaces, or `none`)
- `cost_tier` (`free-install` / `api-key-required` / `paid-service`)
- `privacy_tier` (`cloud` / `local` / `hybrid`)
- `docs_urls` (array)

An audit agent should:

1. Parse every `clawhub-recommendations/*.md` frontmatter (skipping `_template.md` and `README.md`).
2. Fetch each `clawhub_url` — confirm the skill still exists, note current download count + version.
3. Check for deprecation / malicious flags (cross-reference `skill-vetter.md` scan).
4. Update `version_last_verified`, `downloads_last_checked`, `last_checked`, and `maintenance_status` in the frontmatter.
5. Flag any file whose status changed (e.g., author removed the skill, vetter flagged it).
6. For files with `author: "<VERIFY>"` or `maintenance_status: unverified`, prioritize resolving the unknowns.
7. Report a summary PR of what moved.

No single file should need to be re-read to audit another. Each recommendation is independently verifiable.
