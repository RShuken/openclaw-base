# Standard Deployment Protocol

Every `deploy-*.md` skill follows this protocol before executing any steps. This is not optional.

## Phase 0: Discovery

Before touching the client's system, understand what you're working with.

### 1. Read the Client Profile

```
Read clients/<name>.md
```

The client profile contains:
- **OS and environment** (macOS/Windows/Linux, shell, architecture)
- **Installed providers** (which AI APIs, which cloud services)
- **Deployed systems** (which deploy skills have already been run)
- **Preferences and constraints** (what they don't want, what they use instead of defaults)
- **Workspace path** (may not be `~/clawd/`)

If the client profile doesn't exist yet, create one during this session.

### 2. Check What Exists

Run the skill-specific "pre-flight checks" to understand current state:
- Is this system already partially or fully deployed?
- Are prerequisites actually met, or do we need to deploy those first?
- Are there existing files/databases that would be overwritten?
- What workspace path is this client using?

### 3. Identify Adaptations

Every skill has an **Adaptation Points** table listing where the default setup can change. Cross-reference these against the client profile:
- Does the client use the default service, or an alternative?
- Are there path differences?
- Are there preference overrides (different schedule, different channel, etc.)?

### 4. Present the Deployment Plan

Before executing any steps, tell the operator:

> **Deploying [System Name] on [client]**
> - Workspace: `[path]`
> - [Key adaptation 1]: using [choice] (default / alternative because [reason])
> - [Key adaptation 2]: using [choice]
> - Prerequisites confirmed: [list]
> - Will create: [files/databases/cron jobs]

Get operator confirmation before proceeding.

## Variables

Skills use these variables throughout their steps. Resolve them from the client profile before executing.

| Variable | Default | Where It Comes From |
|----------|---------|-------------------|
| `${WORKSPACE}` | `~/clawd/` | Client profile: workspace path |
| `${OPENCLAW_CONFIG}` | `~/.openclaw/` | Client profile: config path |
| `${TELEGRAM_GROUP}` | (none) | Client profile: messaging config |
| `${NOTIFICATION_CHANNEL}` | Telegram | Client profile: preferred notification service |
| `${BACKUP_DESTINATION}` | Google Drive | Client profile: cloud storage preference |
| `${EMAIL_PROVIDER}` | Gmail via gog | Client profile: email service |
| `${CALENDAR_PROVIDER}` | Google Calendar via gog | Client profile: calendar service |
| `${TIMEZONE}` | America/Los_Angeles | Client profile: timezone |

## After Deployment

1. **Update the client profile** (`clients/<name>.md`) with what was deployed, any config values, and any issues encountered.
2. **Run the skill's verification steps** to confirm everything works.
3. **Note any follow-up** needed (e.g., "deploy X next for full functionality").
