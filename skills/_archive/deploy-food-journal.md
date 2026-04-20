# Deploy Food & Symptom Tracking

## Purpose

Install a Telegram-based food and symptom journal. Supports food, drink, symptom (with severity), and note entries. Includes reminders and weekly correlation analysis.

**When to use:** When the user wants to track food intake and correlate with symptoms.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "ls ${WORKSPACE}/memory/health/ 2>/dev/null"
```
- Does the client want health tracking?
- What are the client's meal times? (Default reminders may not fit)
- Does the client have specific tracking needs (allergies, conditions, diet)?

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Alternative | When to Adapt |
|-----------|---------|-------------|---------------|
| Entry types | Food, drink, symptom, note | Add: exercise, sleep, medication, mood | Client tracks more than food/symptoms |
| Severity scale | 1-5 for symptoms | Custom scale, descriptive labels | Client preference |
| Reminder times | 8am, 1pm, 7pm PST | Custom times matching client's meal schedule | Different timezone or eating schedule |
| Storage format | Markdown + JSON by date | JSON only, or database | Client preference for data format |
| Input channel | Telegram Health topic | Discord, Slack, SMS, web form | Client's preferred input method |
| Weekly analysis | Automatic correlation (food → symptoms) | Manual only, or more frequent | Client preference and data volume |
| Correlation triggers | Food → symptom patterns | Additional: sleep → energy, exercise → mood | Client tracks additional dimensions |

## Prerequisites

- `deploy-messaging-setup.md` completed (Telegram Health topic)

## What Gets Installed

### Food Journal System

| Component | Description |
|-----------|-------------|
| Entry types | 4 types: food, drink, symptom, note |
| Severity scale | Symptoms rated 1-5 |
| Reminders | 3x daily: 8am, 1pm, 7pm PST |
| Storage | Markdown + JSON by date |
| Analysis | Weekly correlation of food to symptom triggers |

### Entry Types

| Type | Example Input | What Gets Logged |
|------|---------------|-----------------|
| Food | "Had chicken salad for lunch" | Food item, meal context, timestamp |
| Drink | "Two cups of coffee this morning" | Drink item, quantity, timestamp |
| Symptom | "Headache severity 3" | Symptom name, severity (1-5), timestamp |
| Note | "Slept poorly last night" | Free-text note, timestamp |

### Storage

| File | Location | Format |
|------|----------|--------|
| Markdown log | `memory/health/food-log.md` | Human-readable daily entries |
| JSON log | `memory/health/food-log.json` | Machine-readable for analysis |

Entries are organized by date. Each day gets its own section in the markdown log and its own object in the JSON log.

### Telegram Integration

| Setting | Value |
|---------|-------|
| Topic | Health (ID: 1207) |
| Input method | Natural language messages in the Health topic |
| Reminders | Gentle prompts to log meals at 8am, 1pm, 7pm PST |

### Reminders

Three daily reminders posted to the Health topic:

| Time (PST) | Message |
|-------------|---------|
| 8:00am | Morning reminder to log breakfast |
| 1:00pm | Afternoon reminder to log lunch |
| 7:00pm | Evening reminder to log dinner |

Reminders are gentle prompts, not aggressive. If the user has already logged for that meal window, the reminder can be skipped or softened.

### Weekly Correlation Analysis

After enough data accumulates (at least 7 days of entries), the system runs a weekly correlation analysis:

- Identifies foods/drinks that correlate with symptom entries
- Looks for patterns (e.g., "headaches tend to follow days with high caffeine intake")
- Reports findings to the Health topic
- Improves over time as more data is collected

## Steps

### 1. Create Health Tracking Directory

```
cmd --session <id> --command "mkdir -p ~/clawd/memory/health"
```

### 2. Initialize Log Files

```
cmd --session <id> --command "echo '# Food & Symptom Log\n' > ~/clawd/memory/health/food-log.md"
cmd --session <id> --command "echo '[]' > ~/clawd/memory/health/food-log.json"
```

### 3. Configure Telegram Health Topic

```
cmd --session <id> --command "openclaw config set food-journal.telegram-topic 1207"
```

### 4. Set Up 3x Daily Reminder Crons

```
cmd --session <id> --command "openclaw cron add --name 'Food Journal Morning Reminder' --schedule '0 8 * * *' --tz 'America/Los_Angeles' --command 'node ~/clawd/scripts/food-journal-reminder.js --meal breakfast'"
cmd --session <id> --command "openclaw cron add --name 'Food Journal Afternoon Reminder' --schedule '0 13 * * *' --tz 'America/Los_Angeles' --command 'node ~/clawd/scripts/food-journal-reminder.js --meal lunch'"
cmd --session <id> --command "openclaw cron add --name 'Food Journal Evening Reminder' --schedule '0 19 * * *' --tz 'America/Los_Angeles' --command 'node ~/clawd/scripts/food-journal-reminder.js --meal dinner'"
```

### 5. Set Up Weekly Correlation Analysis

```
cmd --session <id> --command "openclaw cron add --name 'Food Journal Weekly Analysis' --schedule '0 10 * * 0' --tz 'America/Los_Angeles' --command 'node ~/clawd/scripts/food-journal-analysis.js'"
```

This runs every Sunday at 10am PST.

### 6. Test a Food Entry

Post in the Health Telegram topic: "Had coffee and toast for breakfast"

The agent should:
1. Parse this as a food entry
2. Log it to `memory/health/food-log.md` and `memory/health/food-log.json`
3. Confirm the entry was logged

### 7. Test a Symptom Entry

Post in the Health Telegram topic: "Mild headache, severity 2"

The agent should:
1. Parse this as a symptom entry with severity 2
2. Log it to both log files
3. Confirm the entry was logged

### 8. Verify Reminders

Wait for the next reminder window (8am, 1pm, or 7pm PST) and confirm a gentle reminder appears in the Health topic.

## Verification

Check that the system is operational:

```
# Verify log files exist
ls -la ~/clawd/memory/health/food-log.md ~/clawd/memory/health/food-log.json

# Verify cron jobs
openclaw cron list | grep "Food Journal"

# Check log content after a few entries
cat ~/clawd/memory/health/food-log.md
```

Expected:
- Both log files exist at the expected paths
- 4 cron jobs: 3 daily reminders + 1 weekly analysis
- After posting entries, they appear in the markdown log with timestamps

Post a test entry in the Telegram Health topic and verify:
1. Entry appears in `food-log.md` with the correct date and details
2. Entry appears in `food-log.json` in the correct format
3. Confirmation message appears in the Health topic

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Entries not saving | File permissions on `memory/health/` | Check that the agent process has write access. Run `chmod 755 ~/clawd/memory/health/`. |
| Reminders not firing | Cron jobs misconfigured | Check `openclaw cron list` for all three reminder jobs. Verify they show the correct times and timezone. |
| Entries going to wrong topic | Topic ID misconfigured | Verify Health topic is 1207 with `openclaw config show food-journal`. |
| Correlation analysis empty | Not enough data | Need at least 7 days of consistent entries. The analysis needs both food and symptom data to find correlations. |
| Symptom severity not parsed | Natural language parsing missed it | Be more explicit: "Headache, severity 3" works better than "slight headache." The parser looks for the word "severity" followed by a number. |
| JSON log corrupted | Concurrent writes | Check for malformed JSON in `food-log.json`. The log should be an array of entry objects. Repair or reinitialize if needed. |
| Duplicate reminders | Multiple cron jobs for same time | Run `openclaw cron list` and remove duplicate entries. Each meal should have exactly one reminder job. |

## Dependencies

- **Requires:** `deploy-messaging-setup.md` (Telegram Health topic, ID: 1207)
- **Independent of:** All other data systems
- **Pairs well with:** The Health topic is dedicated to this system. No other deploy skill writes to Health topic 1207.
