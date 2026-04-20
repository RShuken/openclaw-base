# Deploy Food & Symptom Tracking

## Compatibility

- **OpenClaw Version**: 2026.4.15+ (compat header rewritten 2026-04-19 — live-verified on 4C)
- **Status**: WORKING — previous "BLOCKED" status was stale; `openclaw cron`, `openclaw config set/get` all exist on 2026.4.15 per `audit/_baseline.md` §3 / §4C.7
- **`openclaw status` does NOT exist** — use `openclaw health` if you need the top-level probe
- **Scheduling**: `openclaw cron add --cron "0 8 * * *"` for breakfast, `"0 13 * * *"` for lunch, `"0 19 * * *"` for dinner; `"0 10 * * SUN"` for weekly correlation roll-up
- **Config**: `openclaw config set channels.telegram.topics.foodJournal <id>` for scalar routing config
- **Model**: agent default (`agents.defaults.model` in `openclaw.json`). Override this skill's LLM routing via env var **`CORRELATION_MODEL`** (see `_authoring/_deploy-common.md` "Per-skill env-var overrides" for the pattern + examples). Applies to the weekly correlation pass.
- **Unique value**: no ClawHub community equivalent exists for scaled food-journal + correlation analysis (per `audit/_baseline.md` §B) — worth keeping

## Purpose

Install a Telegram-based food and symptom journal that tracks food, drink, symptom (with severity), and note entries. Includes configurable daily reminders and weekly correlation analysis to identify food-symptom patterns over time.

**When to use:** When the client wants to track food intake and correlate with symptoms (allergies, digestive issues, energy levels, etc.).

**What this skill does:**
1. Creates the health tracking directory and initializes log files
2. Configures the Telegram Health topic for input
3. Sets up 3x daily meal reminders via cron
4. Schedules weekly correlation analysis
5. Verifies end-to-end with a test entry

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

Resolve these before executing. Sources are listed for each.

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${FOOD_JOURNAL_TOPIC_ID}` | Created during messaging setup (`deploy-messaging-setup.md`), stored in client profile | `1207` |
| `${TIMEZONE}` | Client profile: timezone | `America/Los_Angeles` |
| `${REMINDER_BREAKFAST}` | Client preference or default | `8` (hour, 24h format) |
| `${REMINDER_LUNCH}` | Client preference or default | `13` (hour, 24h format) |
| `${REMINDER_DINNER}` | Client preference or default | `19` (hour, 24h format) |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**

Check if the health tracking directory already exists:

**Remote:**
```
ls ${WORKSPACE}/memory/health/ 2>/dev/null || echo 'NOT_FOUND'
```

Check if log files already have content:

**Remote:**
```
wc -l ${WORKSPACE}/memory/health/food-log.md 2>/dev/null | tr -d ' '
```

Check if food journal cron jobs already exist:

**Remote:**
```
openclaw cron list 2>/dev/null | grep -i "Food Journal" || echo 'NO_CRONS'
```

**Questions to resolve before starting:**
- Does the client want health tracking?
- What are the client's typical meal times? (Default reminders: 8am, 1pm, 7pm)
- Does the client have specific tracking needs (allergies, conditions, diet)?
- What timezone is the client in?

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Entry types | Food, drink, symptom, note | Client tracks more dimensions (exercise, sleep, medication, mood) |
| Severity scale | 1-5 for symptoms | Client prefers custom scale or descriptive labels |
| Reminder times | 8am, 1pm, 7pm | Client has a different eating schedule or timezone |
| Storage format | Markdown + JSON by date | Client prefers JSON only, or a database |
| Input channel | Telegram Health topic | Client uses Discord, Slack, or another channel |
| Weekly analysis | Automatic Sunday 10am correlation | Client wants more/less frequent analysis |
| Correlation triggers | Food-to-symptom patterns | Client wants sleep-to-energy, exercise-to-mood, etc. |

## Prerequisites

- OpenClaw installed and running (`openclaw/install.md`)
- `deploy-messaging-setup.md` completed (Telegram Health topic exists)

## What Gets Installed

| Component | Description |
|-----------|-------------|
| Health directory | `${WORKSPACE}/memory/health/` |
| Markdown log | `${WORKSPACE}/memory/health/food-log.md` — human-readable daily entries |
| JSON log | `${WORKSPACE}/memory/health/food-log.json` — machine-readable for analysis |
| Telegram config | Food journal bound to Health topic `${FOOD_JOURNAL_TOPIC_ID}` |
| Reminder crons | 3x daily meal reminders at configurable times |
| Analysis cron | Weekly correlation analysis (Sunday 10am) |

### Entry Types

| Type | Example Input | What Gets Logged |
|------|---------------|-----------------|
| Food | "Had chicken salad for lunch" | Food item, meal context, timestamp |
| Drink | "Two cups of coffee this morning" | Drink item, quantity, timestamp |
| Symptom | "Headache severity 3" | Symptom name, severity (1-5), timestamp |
| Note | "Slept poorly last night" | Free-text note, timestamp |

## Steps

### Phase 1: Create Storage

#### 1.1 Create Health Tracking Directory `[AUTO]`

Create the directory structure for health data storage.

**Remote:**
```
mkdir -p ${WORKSPACE}/memory/health
```

Expected: Directory exists at `${WORKSPACE}/memory/health/`.

If this fails: Check that `${WORKSPACE}/memory/` exists. If the workspace itself is missing, run `openclaw/install.md` first.

If already exists: Skip. `mkdir -p` is idempotent.

#### 1.2 Initialize the Markdown Log File `[AUTO]`

Create the human-readable food log with a header. Uses `printf` to correctly output the trailing newline.

**Remote:**
```
printf '# Food & Symptom Log\n\n' > ${WORKSPACE}/memory/health/food-log.md
```

Expected: File exists at `${WORKSPACE}/memory/health/food-log.md` with the header line.

If this fails: Check write permissions on the health directory.

If already exists with entries: Do NOT overwrite. Skip this step if the file already has content beyond the header (check line count from pre-flight). Only re-initialize if the file is missing or empty.

#### 1.3 Initialize the JSON Log File `[AUTO]`

Create the machine-readable log as an empty JSON array.

**Remote:**
```
printf '[]' > ${WORKSPACE}/memory/health/food-log.json
```

Expected: File exists at `${WORKSPACE}/memory/health/food-log.json` containing `[]`.

If this fails: Check write permissions on the health directory.

If already exists with entries: Do NOT overwrite. Skip this step if the file already has valid JSON content beyond an empty array. Only re-initialize if the file is missing, empty, or contains invalid JSON.

### Phase 2: Configure Telegram Integration

#### 2.1 Set Telegram Health Topic `[GUIDED]`

Bind the food journal to the Health topic in Telegram so the agent knows where to listen for food/symptom entries and where to send reminders.

**Remote:**
```
openclaw config set food-journal.telegram-topic ${FOOD_JOURNAL_TOPIC_ID}
```

Expected: Configuration updated. Verify with `openclaw config show food-journal`.

If this fails: Check that `openclaw config` command is available. If the config key doesn't exist, the OpenClaw version may need updating, or the food journal skill may need to be registered first.

If already configured: Check the current value. If it matches `${FOOD_JOURNAL_TOPIC_ID}`, skip. If different, update to the correct value.

### Phase 3: Schedule Reminders

#### 3.1 Set Up Morning Reminder `[AUTO]`

Schedule a breakfast logging reminder. The reminder sends a gentle prompt to the Health topic.

**Remote:**
```
openclaw cron add --name 'Food Journal Morning Reminder' --schedule '0 ${REMINDER_BREAKFAST} * * *' --tz '${TIMEZONE}' --command 'node ${WORKSPACE}/scripts/food-journal-reminder.js --meal breakfast'
```

Expected: Cron job appears in `openclaw cron list` output with the correct schedule and timezone.

If this fails: Verify `openclaw cron add` syntax. Check that the scripts directory and reminder script exist.

If already exists: Check the existing schedule matches. If a "Food Journal Morning Reminder" cron already exists with the correct time and timezone, skip. If times differ, remove the old one and re-add.

#### 3.2 Set Up Afternoon Reminder `[AUTO]`

Schedule a lunch logging reminder.

**Remote:**
```
openclaw cron add --name 'Food Journal Afternoon Reminder' --schedule '0 ${REMINDER_LUNCH} * * *' --tz '${TIMEZONE}' --command 'node ${WORKSPACE}/scripts/food-journal-reminder.js --meal lunch'
```

Expected: Cron job appears in `openclaw cron list` with the correct schedule.

If this fails: Same checks as 3.1.

If already exists: Same idempotency check as 3.1.

#### 3.3 Set Up Evening Reminder `[AUTO]`

Schedule a dinner logging reminder.

**Remote:**
```
openclaw cron add --name 'Food Journal Evening Reminder' --schedule '0 ${REMINDER_DINNER} * * *' --tz '${TIMEZONE}' --command 'node ${WORKSPACE}/scripts/food-journal-reminder.js --meal dinner'
```

Expected: Cron job appears in `openclaw cron list` with the correct schedule.

If this fails: Same checks as 3.1.

If already exists: Same idempotency check as 3.1.

### Phase 4: Schedule Weekly Analysis

#### 4.1 Set Up Weekly Correlation Analysis `[AUTO]`

Schedule the weekly food-symptom correlation analysis. This runs every Sunday at 10am in the client's timezone. It analyzes the past week's entries to find patterns between food/drink intake and symptom occurrences.

**Remote:**
```
openclaw cron add --name 'Food Journal Weekly Analysis' --schedule '0 10 * * 0' --tz '${TIMEZONE}' \
  --env CORRELATION_MODEL=openai-codex/gpt-5.4-nano \
  --command 'node ${WORKSPACE}/scripts/food-journal-analysis.js'
```

`CORRELATION_MODEL` routes the weekly correlation analysis LLM call. Omit to defer to the agent default. In `scripts/food-journal-analysis.js`:

```javascript
const MODEL = process.env.CORRELATION_MODEL || '';  // empty → agent default
const modelFlag = MODEL ? ['--model', MODEL] : [];
spawn('openclaw', ['agent', '--agent', 'main', ...modelFlag, '-m', prompt]);
```

Expected: Cron job appears in `openclaw cron list`. The analysis requires at least 7 days of entries with both food and symptom data before it produces meaningful results.

If this fails: Verify the analysis script exists at `${WORKSPACE}/scripts/food-journal-analysis.js`. If missing, the script needs to be created as part of this deployment (see Phase 5 note).

If already exists: Check the schedule matches. If a "Food Journal Weekly Analysis" cron already exists with the same schedule, skip.

### Phase 5: Verify

#### 5.1 Test a Food Entry `[GUIDED]`

Post a test message in the Telegram Health topic to verify end-to-end logging.

Post in the Health Telegram topic (topic ID `${FOOD_JOURNAL_TOPIC_ID}`): "Had coffee and toast for breakfast"

Expected: The agent should:
1. Parse this as a food entry
2. Log it to `${WORKSPACE}/memory/health/food-log.md` and `food-log.json`
3. Confirm the entry was logged in the Health topic

If this fails: Check that the agent is running (`openclaw health`), the topic ID is correct (`openclaw config show food-journal`), and the agent has write access to the log files.

#### 5.2 Test a Symptom Entry `[GUIDED]`

Post another test message to verify symptom parsing.

Post in the Health Telegram topic: "Mild headache, severity 2"

Expected: The agent should:
1. Parse this as a symptom entry with severity 2
2. Log it to both log files with the severity value
3. Confirm the entry was logged

If this fails: The symptom parser looks for the word "severity" followed by a number. More explicit phrasing ("Headache, severity 3") works better than ambiguous phrasing ("slight headache").

#### 5.3 Verify All Cron Jobs `[AUTO]`

Confirm all four cron jobs are registered correctly.

**Remote:**
```
openclaw cron list | grep "Food Journal"
```

Expected: Four entries:
- Food Journal Morning Reminder
- Food Journal Afternoon Reminder
- Food Journal Evening Reminder
- Food Journal Weekly Analysis

If this fails: Re-run the cron add commands from Phases 3 and 4. Check for duplicate entries and remove them.

#### 5.4 Verify Log File Content `[AUTO]`

After the test entries, confirm data was written to both log files.

**Remote:**
```
cat ${WORKSPACE}/memory/health/food-log.md
```

**Remote:**
```
cat ${WORKSPACE}/memory/health/food-log.json
```

Expected: The markdown log contains the test entries with timestamps. The JSON log contains corresponding entry objects.

If this fails: Check file permissions (`ls -la ${WORKSPACE}/memory/health/`). Ensure the agent process has write access.

## Verification

Run these checks to confirm the full system is operational:

**Remote:**
```
ls -la ${WORKSPACE}/memory/health/food-log.md ${WORKSPACE}/memory/health/food-log.json
```

**Remote:**
```
openclaw cron list | grep "Food Journal"
```

**Remote:**
```
openclaw config show food-journal
```

Expected:
- Both log files exist at the expected paths
- 4 cron jobs: 3 daily reminders + 1 weekly analysis
- Food journal config shows topic ID `${FOOD_JOURNAL_TOPIC_ID}`
- After posting entries, they appear in the markdown log with timestamps

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Entries not saving | File permissions | Check agent has write access: `ls -la ${WORKSPACE}/memory/health/` |
| Reminders not firing | Cron jobs misconfigured | Check `openclaw cron list` for all three reminder jobs. Verify correct times and timezone. |
| Entries going to wrong topic | Topic ID mismatch | Verify with `openclaw config show food-journal`. Should show `${FOOD_JOURNAL_TOPIC_ID}`. |
| Correlation analysis empty | Not enough data | Need at least 7 days of entries with both food AND symptom data. |
| Symptom severity not parsed | Ambiguous phrasing | Use explicit format: "Headache, severity 3" rather than "slight headache." |
| JSON log corrupted | Concurrent writes | Check for malformed JSON. Repair or reinitialize as empty array `[]`. |
| Duplicate reminders | Multiple cron jobs for same time | Run `openclaw cron list`, remove duplicates. Each meal should have exactly one reminder. |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1-1.3 Create storage | Execute silently | Execute silently | Confirm before creating files |
| 2.1 Configure Telegram topic | Execute silently | Confirm topic ID before setting | Confirm topic ID before setting |
| 3.1-3.3 Schedule reminders | Execute silently | Confirm reminder times with client | Confirm each cron command |
| 4.1 Schedule weekly analysis | Execute silently | Confirm schedule | Confirm command |
| 5.1-5.2 Test entries | Execute, report results | Execute, wait for client confirmation | Walk through each test with client |
| 5.3-5.4 Verify crons and logs | Execute, report results | Execute, report results | Show output, confirm |

## Dependencies

- **Depends on:** `openclaw/install.md` (OpenClaw must be installed), `deploy-messaging-setup.md` (Telegram Health topic must exist)
- **Independent of:** All other data systems
- **Pairs well with:** The Health topic is dedicated to this system. No other deploy skill writes to the Health topic.
