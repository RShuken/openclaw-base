# HEARTBEAT (template)

Recurring pulses the agent runs without being asked. The agent's "involuntary nervous system."

Replace every `{{...}}` before deploying. Each pulse defines: when it fires, what it does, what success looks like, who/what it notifies on failure.

## Pulse: morning-briefing

- **When:** {{CRON or e.g., "weekdays 7:00am local time"}}
- **What:** {{e.g., "summarize calendar + overnight email + new CRM activity, post to Telegram"}}
- **Success looks like:** A formatted message in the briefing channel, no errors in `~/.openclaw/logs/heartbeat-YYYY-MM-DD.log`.
- **On failure:** Telegram alert to {{OWNER_HANDLE}}.

## Pulse: hourly-git-autosync

- **When:** Top of every hour
- **What:** Commit + push tracked workspace dirs (see `deploy-git-autosync-v2.md`).
- **Success:** Latest commit on `main` is < 60 min old.
- **On failure:** Skip silently if no changes; alert if push fails twice in a row.

## Pulse: nightly-security-council

- **When:** Daily 3:00am local
- **What:** {{see `deploy-security-council-v2.md`}}
- **Success:** Report file in `~/.openclaw/security/YYYY-MM-DD.md`.
- **On failure:** Alert; do NOT auto-remediate.

## Pulse: weekly-health-check

- **When:** Sundays 9:00am local
- **What:** {{see `deploy-health-monitoring.md`}}
- **Success:** Silent (silent = healthy).
- **On failure:** Telegram alert with which check failed.

## Pulse Discipline

- Heartbeats run as standalone scripts via `launchd` (or `cron add`), NOT inside long-lived openclaw sessions.
- Every pulse logs to `~/.openclaw/logs/heartbeat-YYYY-MM-DD.log` with start time, end time, exit status.
- A pulse that runs longer than its interval is a bug — fix the pulse, don't extend the interval.
- Heartbeats never modify external state without an approval gate (see IDENTITY).
