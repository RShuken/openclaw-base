# Deploy Health Monitoring Heartbeat

## Compatibility

- **OpenClaw Version**: 2026.4.15+ (rewritten 2026-04-20 per audit/_baseline.md §O community anti-pattern)
- **Status**: WORKING
- **Scheduling**: single `openclaw cron add` job, runs once daily (cheap checks first; escalate only on alert per §L)
- **Model**: agent default (`agents.defaults.model` in `openclaw.json`). No LLM call unless a check fires. Override the triage-pass LLM routing via env var **`HEARTBEAT_TRIAGE_MODEL`** (see `_authoring/_deploy-common.md` "Per-skill env-var overrides" for the pattern + examples).
- **Silent = healthy**: no daily "all clear" noise. Alerts only when something's wrong.

## Purpose

Install a single daily health probe that runs fast bash checks across the client's deployed systems, writes structured results to `heartbeat-state.json`, and notifies only when a check fails. Philosophy: cheap checks by default, LLM analysis only when triage is needed.

**When to use:** After the core workspace is set up. Provides ongoing operational oversight without daily noise.

**What this skill does:**
1. Installs a single `heartbeat.sh` script with 8 independent check functions (disk, repo size, gateway binding, gateway auth, cron-job failures, git backup, memory scan, social freshness)
2. Schedules it via `openclaw cron add` at 07:00 in the client's timezone
3. Creates `heartbeat-state.json` for result tracking
4. Pipes alerts to the configured notification channel via `openclaw channels`
5. On `HEARTBEAT_ALERT`, the checked skill can optionally hand off a Markdown summary to the agent's default model for operator-facing triage — but only then

## Why this redesign

Prior version (pre-2026-04-20) fired **3 separate cron jobs** (every-30-min / weekly / nightly). That fan-out is the exact anti-pattern documented in `audit/_baseline.md` §O: *"Don't fan out N cron jobs — collapse into heartbeat."* Cost was ~5× higher than collapsed form (each cron spawns an isolated context the model re-reads cold). Also: `agents.defaults.heartbeat.model` override is broken (GitHub openclaw/openclaw#9556 per §M), so routing heartbeat to a cheap model via config doesn't work — the only reliable way is bash-first with LLM escalation as a per-finding tool call.

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`.

## Variables

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | `agents.defaults.workspace` | `~/.openclaw/workspace` |
| `${TIMEZONE}` | Client profile | `America/Los_Angeles` |
| `${DISK_WARNING_GB}` | Adaptation | `20` |
| `${DISK_URGENT_GB}` | Adaptation | `10` |
| `${REPO_SIZE_MB}` | Adaptation | `500` |
| `${NOTIFICATION_CHANNEL}` | `channels.*` in openclaw.json | `telegram` |
| `${HEALTH_TOPIC_ID}` | Messaging setup | `1842` |

## Before You Start

Run the Phase 0.5 compat check per `_authoring/_deploy-common.md`.

**Pre-flight checks specific to this skill:**

```
openclaw cron list 2>&1 | grep -i heartbeat
```
Expected: no existing `heartbeat` job, or a single one. If you find legacy entries (`Cron Health Check`, `Disk Space Check`, `Security Review`), remove them — this rewrite replaces all three.

```
ls ${WORKSPACE}/memory/heartbeat-state.json 2>/dev/null
```
Expected: file path printed, or empty (first install).

```
openclaw doctor 2>&1 | head -20
```
Expected: doctor output. Any failing probes here represent **findings the health skill doesn't need to duplicate** — it extends `doctor`, doesn't replace it.

## Adaptation Points

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Schedule | Daily 07:00 `${TIMEZONE}` | Client wants different time; weekly cadence OK for low-change systems |
| Disk thresholds | Warning 20GB, urgent 10GB | Smaller/larger disk |
| Repo size threshold | 500MB | Large monorepos |
| Active checks | All 8 | Skip checks for systems not deployed (e.g. no social tracker → skip social freshness) |
| Alert channel | `${NOTIFICATION_CHANNEL}` via `openclaw channels` | Client uses a different channel |
| LLM triage on alert | Enabled | Disable for pure-ops mode (silent-fail friendly) |

## Prerequisites

- OpenClaw 2026.4.15+ (`openclaw --version`)
- `deploy-messaging-setup.md` completed (alert delivery channel exists)
- Optional: `deploy-social-tracking.md`, `deploy-git-autosync.md` — enable corresponding sub-checks if installed

## What Gets Installed

| Component | Location | Purpose |
|-----------|----------|---------|
| `heartbeat.sh` | `${WORKSPACE}/scripts/heartbeat.sh` | Single script running all 8 checks |
| `heartbeat-state.json` | `${WORKSPACE}/memory/heartbeat-state.json` | Result tracking (timestamps, status, numbers) |
| One cron job | OpenClaw cron store | Runs `heartbeat.sh` daily at 07:00 |

### The 8 checks

All run in one bash script with shared output. Each function prints `OK` or `ALERT <code>: <detail>` to stdout; the orchestrator aggregates and decides whether to notify.

| # | Check | What | Runtime |
|---|-------|------|---------|
| 1 | `check_disk_space` | `df` output vs. `${DISK_WARNING_GB}` / `${DISK_URGENT_GB}` | <50ms |
| 2 | `check_repo_size` | `du -sh ${WORKSPACE}` vs. `${REPO_SIZE_MB}` | <200ms |
| 3 | `check_gateway_binding` | `lsof -i :18789` — must be 127.0.0.1 only | <100ms |
| 4 | `check_gateway_auth` | `openclaw config get gateway.auth.mode` must not be `none` | <500ms |
| 5 | `check_cron_failures` | `openclaw cron runs --since 24h --status failed` count | <500ms |
| 6 | `check_git_backup` | Workspace git tree: `git log -1 --format=%ct` age check | <100ms |
| 7 | `check_memory_scan` | Weekly only (skip on other days): grep memory files for `ignore previous instructions` / `</system>` / known injection patterns | <2s |
| 8 | `check_social_freshness` | Skip if no `social-tracker/` dir. Otherwise `stat` newest JSONL mtime | <50ms |

Total budget: **< 4 seconds.** No LLM calls in the default path.

## Steps

### Phase 1: Create state infrastructure `[AUTO]`

```
mkdir -p ${WORKSPACE}/memory ${WORKSPACE}/scripts
test -f ${WORKSPACE}/memory/heartbeat-state.json || printf '{}' > ${WORKSPACE}/memory/heartbeat-state.json
```

### Phase 2: Install `heartbeat.sh` `[GUIDED]`

Write the script to `${WORKSPACE}/scripts/heartbeat.sh` using the base64 file-transfer pattern from `_authoring/_deploy-common.md`. The script structure:

```bash
#!/usr/bin/env bash
set -u
WS="${WORKSPACE:-$HOME/.openclaw/workspace}"
STATE="$WS/memory/heartbeat-state.json"
DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)
DOW=$(date +%u)  # 1-7, Monday=1
ALERTS=()

check_disk_space() { ... prints "OK" or "ALERT disk_urgent: 8GB free" ... }
check_repo_size()  { ... }
check_gateway_binding() { ... }
check_gateway_auth() { ... }
check_cron_failures() { ... }
check_git_backup() { ... }
check_memory_scan() { [ "$DOW" = "7" ] || return 0; ...  }  # Sunday only
check_social_freshness() { [ -d "$WS/social-tracker" ] || return 0; ...  }

for fn in check_disk_space check_repo_size check_gateway_binding \
          check_gateway_auth check_cron_failures check_git_backup \
          check_memory_scan check_social_freshness; do
  RESULT=$($fn 2>&1)
  if [[ "$RESULT" == ALERT* ]]; then ALERTS+=("$RESULT"); fi
done

# Update state file atomically
python3 - "$STATE" "$DATE" "${ALERTS[@]}" <<'PY'
import json, sys, pathlib
path, ts, *alerts = sys.argv[1:]
state = json.loads(pathlib.Path(path).read_text() or "{}")
state["last_run"] = ts
state["alerts"] = alerts
pathlib.Path(path).write_text(json.dumps(state, indent=2))
PY

# Alert only if findings
if [ ${#ALERTS[@]} -gt 0 ]; then
  MSG=$(printf 'HEARTBEAT_ALERT on %s\n%s' "$(hostname)" "$(printf '%s\n' "${ALERTS[@]}")")
  openclaw channels send --channel "${NOTIFICATION_CHANNEL}" --to "${HEALTH_TOPIC_ID}" --text "$MSG"
  # Optional: hand off to LLM for triage — only on alert, only once per day
  if [ "${HEARTBEAT_LLM_TRIAGE:-1}" = "1" ]; then
    MODEL_FLAG=""
    if [ -n "${HEARTBEAT_TRIAGE_MODEL:-}" ]; then
      MODEL_FLAG="--model ${HEARTBEAT_TRIAGE_MODEL}"
    fi
    echo "$MSG" | openclaw agent --agent main $MODEL_FLAG -m "Triage these health findings. Suggest the single highest-priority action." --timeout 60
  fi
else
  echo "HEARTBEAT_OK"
fi
exit 0
```

Then `chmod +x ${WORKSPACE}/scripts/heartbeat.sh`.

**Idempotency:** If `heartbeat.sh` already exists, diff before overwriting. Backup as `.bak` on mismatch.

### Phase 3: Schedule the single cron job `[GUIDED]`

Remove any legacy 3-job setup, then add the single daily job:

```
for job in 'Cron Health Check' 'Disk Space Check' 'Security Review'; do
  openclaw cron rm --name "$job" 2>/dev/null || true
done
openclaw cron add \
  --name 'heartbeat' \
  --cron '0 7 * * *' \
  --tz "${TIMEZONE}" \
  --session isolated \
  --env HEARTBEAT_TRIAGE_MODEL=openai-codex/gpt-5.4-nano \
  --command "${WORKSPACE}/scripts/heartbeat.sh"
```

Verify:
```
openclaw cron list | grep heartbeat
openclaw cron status | grep heartbeat
```

Expected: exactly one `heartbeat` job, scheduled for 07:00 in `${TIMEZONE}`.

### Phase 4: Baseline run `[AUTO]`

Run it once manually to populate state:

```
${WORKSPACE}/scripts/heartbeat.sh
cat ${WORKSPACE}/memory/heartbeat-state.json | python3 -m json.tool
```

Expected: either `HEARTBEAT_OK` + state file showing `"alerts": []`, OR `HEARTBEAT_ALERT` + a Telegram (or other channel) message listing the specific findings.

## Verification

```
openclaw cron list | grep -c heartbeat   # Expect: 1
openclaw cron runs --name heartbeat --limit 1   # Expect: 1 recent run
cat ${WORKSPACE}/memory/heartbeat-state.json | python3 -c 'import sys,json; d=json.load(sys.stdin); print("last_run:", d.get("last_run")); print("alert_count:", len(d.get("alerts",[])))'
```

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Two or three cron jobs running | Legacy 3-job setup not fully removed | Re-run Phase 3 `openclaw cron rm` loop. Verify with `openclaw cron list \| grep -iE 'health\|disk\|security\|heartbeat'` |
| No `heartbeat` cron after add | `openclaw cron` not available | Verify `openclaw --version` is 2026.4.15+. If older, upgrade — do not fall back to system crontab (that's the old anti-pattern this rewrite replaces) |
| `channels send` returns error | `deploy-messaging-setup.md` didn't complete | Verify `openclaw channels list` shows the target channel. If `openclaw channels` isn't available, fall back to raw Bot API via curl |
| False-positive disk alert | `df` reports free space for wrong mount | Check `df /` explicitly vs the volume holding `${WORKSPACE}` |
| Gateway binding alert on localhost | `lsof` output parsing sensitive to locale | Pin `LC_ALL=C lsof -iTCP:18789 -sTCP:LISTEN` in the check |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| Phase 1 (state) | silent | confirm | confirm each |
| Phase 2 (script install) | silent | confirm before install | confirm each |
| Phase 3 (schedule) | silent | confirm | confirm |
| Phase 4 (baseline) | silent | confirm | confirm |

## Dependencies

- **Depends on:** `deploy-messaging-setup.md` (for alert channel)
- **Enhanced by:** `deploy-social-tracking.md`, `deploy-git-autosync-v2.md` (enable their sub-checks if installed; skip gracefully if not)
- **Replaces:** the earlier multi-cron version (3 separate jobs). Do NOT deploy both — they will double-count cron-failure alerts.
