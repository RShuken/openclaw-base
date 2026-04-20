# BOOTSTRAP (template)

Loaded **only on the first session after OpenClaw install**. After the first run, delete or move this file so it doesn't load again.

Replace every `{{...}}` before deploying. Target: 40-80 lines.

---

## Purpose

On the agent's very first session, it needs to confirm its identity is set up correctly and do any one-time workspace setup that can't happen purely at install time (needs human confirmation).

## First-session checklist

When the agent boots for the first time, it should:

1. **Confirm identity** — read IDENTITY.md + SOUL.md, introduce itself to the owner in character. Ask: "Is this right? Anything to adjust?"
2. **Verify the owner profile** — read USER.md, confirm the facts with the operator. Ask about anything marked `{{...}}` that wasn't filled in.
3. **Verify tool access** — probe each tool listed in TOOLS.md. Report which ones work and which need auth / re-auth.
4. **Set up the first memory entry** — write a short `workspace/memory/YYYY-MM-DD.md` noting "Agent deployed today. Initial tool probes: [results]."
5. **Ask about preferences** — anything the operator wants to change in AGENTS.md approval gates, HEARTBEAT.md cadence, or SOUL.md voice?
6. **Mark bootstrap complete** — once the above is done, rename this file to `BOOTSTRAP-completed-{{DATE}}.md` so the agent knows not to re-run.

## Topics to cover with the owner

The agent should walk through these with the operator during the first session. Mark each as the operator confirms.

- [ ] Agent identity (name, emoji, voice) — from IDENTITY.md + SOUL.md
- [ ] Primary messaging channel — from TOOLS.md / channels config
- [ ] Working hours and timezone — from USER.md
- [ ] Approval gates — from AGENTS.md (what needs explicit approval)
- [ ] Hard nos — from USER.md (what the agent should never do)
- [ ] Heartbeat cadence — from HEARTBEAT.md (what runs automatically, how often)
- [ ] Memory backend — which option from `skills/memory-options/` is installed

## Checks to run

```bash
# Confirm core subsystems
openclaw health
openclaw doctor

# Confirm auth profiles
openclaw models auth list

# Confirm memory is indexed
openclaw memory status --json

# Confirm at least one channel is reachable
openclaw channels list
```

## After bootstrap

Rename this file so it doesn't load on subsequent sessions:

```bash
mv ~/.openclaw/workspace/BOOTSTRAP.md ~/.openclaw/workspace/BOOTSTRAP-completed-$(date +%Y-%m-%d).md
```

The agent should propose this rename at the end of the first session. The operator confirms before it happens.

## If bootstrap is interrupted

If the first session ends before bootstrap is complete, leave BOOTSTRAP.md in place and pick up where you left off on the next session. Record progress in the first `workspace/memory/YYYY-MM-DD.md` file so the agent knows which checklist items are done.

## Do not put here

- Permanent configuration — that goes in openclaw.json or the relevant template file
- Long-term memory — that's MEMORY.md
- Recurring tasks — that's HEARTBEAT.md or `openclaw cron`
