# Enterprise v2 Reference Skills

Sanitized snapshot of a v2 skill set originally written for a venture-capital-firm engagement. Kept here as **reference material** — examples of refined, opinionated takes on common skills — but **NOT the default** for `openclaw-base` deployments.

## Scrubbed — all identifiers are placeholders

Before publishing, client names, emails, chat IDs, employee names, and voice samples were replaced with generic placeholders:

| Placeholder | Means |
|-------------|-------|
| `{{COMPANY_NAME}}` | Client company / firm |
| `{{COMPANY_DOMAIN}}` | Company email domain |
| `{{COMPANY_SLUG}}` | Lowercase slug (paths / URLs) |
| `{{PRINCIPAL_NAME}}` | The senior operator (e.g. MD, Founder) |
| `{{PRINCIPAL_EMAIL}}` / `{{PRINCIPAL_CHAT_ID}}` | Their email + Telegram ID |
| `{{AGENT_NAME}}` | The agent's persona name |
| `{{EA_NAME}}` | Executive assistant |
| `{{TEAM_ANALYST}}` / `{{TEAM_MEMBER_1..N}}` | Team members by role |
| `{{FAMILY_NAME}}` / `{{FAMILY_MEMBER_1}}` | Family members (used where personal life intersects work) |
| `{{FAMILY_FOUNDATION}}` | Family-controlled entity |
| `{{EXTERNAL_CONTACT_1..N}}` | External people (example contacts in meeting lists) |
| `{{OPERATOR_TELEGRAM_USER}}` / `{{OPERATOR_TELEGRAM_ID}}` | The installing operator's Telegram |

Fill these in when adapting a skill for a new engagement.

## Why Reference-Only

These drafts are tuned for a venture capital firm's specific shape:

- Notion-as-CRM (not SQLite) with 20K+ Persons/Company directories
- Notion-dashboard approval flows (not Telegram quick-actions)
- Human-in-the-loop on every external action
- VC-flavored briefings (deal flow, portfolio companies, board meetings)
- A multi-person team with EA routing patterns
- Pre-Codex model assumptions (some files still hardcode Claude Opus — update per `skills/deploy-model-routing.md` when porting)

They're heavier and more conservative than the base library's defaults. For generic `openclaw-base` deployments, prefer the lighter Codex-friendly top-level skills.

## How To Use This Folder

- **Browse** when designing a new skill to see how a v2 was structured, what edge cases were handled
- **Cherry-pick** patterns (idempotency checks, verification steps, troubleshooting tables)
- **Do not** symlink or import wholesale — engagement-specific assumptions will break a generic deployment
- **When adapting:** copy the file out, replace every `{{...}}` placeholder with real values, update model IDs against the routing skill, review approval gates for fit

## Origin

Snapshot dated 2026-04-19. Scrubbed for public release 2026-04-20.
