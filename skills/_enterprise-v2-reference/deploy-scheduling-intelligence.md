# Deploy Scheduling Intelligence

> **📌 REFERENCE SNAPSHOT — NOT FOR DEFAULT DEPLOYMENT.** This file was preserved from the {{PRINCIPAL_NAME}} / {{COMPANY_NAME}} engagement as a reference example of how a custom office/VC deployment was built. **Do NOT install this skill as part of a generic OpenClaw deployment.** It may contain engagement-specific defaults (emails, chat IDs, team names, VC-flavored workflows), Claude Opus routing (we default to Codex per `audit/_baseline.md` §BB in the parent repo), or stale "Blocked Commands" claims from 2026.2.26 that were resolved in 2026.4.15+. Use as inspiration when building custom-office skills, not as a default.

## Compatibility
- **OpenClaw Version**: 2026.3.27+
- **Status**: DRAFT
- **Layer**: 1 (Communication) — works alongside Calendar integration
- **Architecture**: Standalone Node.js + `gog` CLI (built-in OpenClaw calendar tool) + `goplaces` CLI (built-in OpenClaw places tool) + Google Maps Directions API + launchd 30-min heartbeat. Leverages existing OpenClaw tooling instead of building from scratch.

## Purpose

Install an intelligent scheduling engine that understands the difference between moveable and fixed meetings, automatically reschedules internal meetings when externals are booked, calculates travel time with live traffic data, and adds parking information to calendar events.

**Why this exists:** {{EA_NAME}}'s #1 pain point (2026-03-30 meeting). Current tools (Calendly, etc.) only see blank spots. {{EA_NAME}} manually classifies which meetings can be moved, calculates drive times, looks up parking, and juggles overlapping requests. This is hours of manual work per week that follows clear rules an AI can learn.

**Why it's needed by other skills:**
- **Daily Briefing** reports scheduling status (moveable vs. fixed meetings, conflicts)
- **Nightman** uses the scheduling rules for overnight calendar enrichment
- **Meeting Prep** needs to know meeting types for dossier depth (external = full prep, internal = light)
- **{{EA_NAME}}'s approval flow** — scheduling changes route through {{EA_NAME}}'s Telegram for approval

**Core scheduling rules (from {{EA_NAME}}):**
1. **Internal meetings** ({{PRINCIPAL_NAME}} + {{TEAM_MEMBER_1}}, {{PRINCIPAL_NAME}} + {{TEAM_MEMBER_2}}, {{PRINCIPAL_NAME}} + {{TEAM_ANALYST}}) — MOVEABLE unless explicitly marked time-sensitive
2. **External meetings** (anyone outside {{COMPANY_NAME}}) — FIXED, never auto-move
3. **When booking over a moveable meeting**: System must find a new time for the displaced meeting on both attendees' calendars, send the reschedule, and only then confirm the new external meeting
4. **Travel time**: Every in-person meeting needs a travel block before AND after. Use Google Maps Directions API with departure_time set to the actual meeting time for traffic-aware estimates. Use the more conservative estimate.
5. **Parking info**: For known locations, auto-add parking details to the travel time block description
6. **No overlapping events**: {{PRINCIPAL_NAME}}'s calendar must never show two events at the same time after scheduling completes

## What Gets Installed

1. **Meeting classification engine**: Rules-based + ML classifier for moveable vs. fixed. Reads attendee list, checks if all attendees are internal ({{COMPANY_NAME}} domain), checks for "time-sensitive" flag.
2. **Travel time calculator**: Google Maps Directions API integration. Calculates drive time for in-person meetings with traffic estimates. Caches common routes.
3. **Parking database**: JSON file of known venue parking info (manually seeded, enriched over time by Nightman).
4. **Rescheduling engine**: When a moveable meeting needs to move, finds the next available slot on all attendees' calendars and proposes the move to {{EA_NAME}} for approval.
5. **{{EA_NAME}} approval flow**: Telegram inline keyboard for scheduling changes. {{EA_NAME}} approves → calendar updated. {{EA_NAME}} rejects → operator notified.

## Shared Scheduling Rules

This skill shares `config/scheduling-rules.json` with Meeting Prep. {{EA_NAME}} owns and approves the rules. The file contains:
- Internal people list with ranks ({{PRINCIPAL_NAME}}=10, {{TEAM_MEMBER_1}}=9, {{TEAM_MEMBER_2}}=7, etc.)
- Meeting type rankings (external=100, internal time-sensitive=90, {{PRINCIPAL_NAME}}+team=40, focus block=10)
- Topic-based overrides (closing=95, board=90, sync=30, etc.)
- Non-moveable hard rules (external attendees, [TS] flag, within 2 hours, already rescheduled)
- Conflict resolution: lower-ranked meeting moves, tie-break by later-scheduled

See `deploy-meeting-prep.md` for the full rules specification and {{EA_NAME}}'s approval flow.

## Built-in OpenClaw Tools Leveraged

| Tool | What | How We Use It |
|------|------|--------------|
| **`gog` CLI** | Google Calendar read/write via OAuth/Service Account | Read events, create travel blocks, detect conflicts, update events |
| **`goplaces` CLI** | Google Places API — location search | Parking lookup near meeting venues, restaurant search, venue details |
| **Google Maps Directions API** | Travel time with traffic | Time-of-day estimates, conservative routing, departure time calculation |
| **launchd** | Native macOS scheduling | 30-minute heartbeat to monitor calendar changes |

`gog` is already installed on {{AGENT_NAME}}. `goplaces` needs installation: `brew install goplaces` + set `GOOGLE_PLACES_API_KEY` in .env.

## Calendar Heartbeat (30-Minute Monitor)

Every 30 minutes, {{AGENT_NAME}} scans {{PRINCIPAL_NAME}}'s calendar for changes and enriches new/modified events:

```
Every 30 min via launchd:

1. Pull current calendar state via `gog calendar events list`
2. Compare against last-known state (SQLite cache)
3. For each NEW or MODIFIED event:
   a. Classify: internal vs. external (from scheduling-rules.json)
   b. Check: virtual? Has Meet/Zoom link? If not → flag {{EA_NAME}}
   c. Check: in-person? Has travel block? If not → calculate + recommend
   d. Check: conflicts with existing events? → recommend resolution
   e. Send enrichment to {{EA_NAME}} via Telegram:

      "📅 New event detected (added by {{EA_NAME}}):
       'Lunch with {{EXTERNAL_CONTACT_1}}' — Tuesday 12:00 PM
       📍 Frasca Food & Wine, Boulder

       I can add:
       ✓ Travel time: 15 min from office (11:40 AM departure)
       ✓ Parking: Street parking on Pine St or Frasca valet
       ✓ Return travel: 15 min (1:30 PM block)

       [Approve All] [Modify] [Skip — I'll handle it]"

4. If {{EA_NAME}} approves → create travel blocks + add parking to description
5. Update calendar state cache
```

**Key principle:** {{AGENT_NAME}} enriches, {{EA_NAME}} approves. {{AGENT_NAME}} never modifies events without {{EA_NAME}}'s OK.

### What Triggers Enrichment

| Trigger | What {{AGENT_NAME}} Does |
|---------|---------------|
| {{EA_NAME}} creates a new event | Detect via heartbeat → classify → enrich → ask {{EA_NAME}} to approve |
| {{PRINCIPAL_NAME}} adds an event via Telegram | Same flow — detect, enrich, approve |
| Event location changes | Recalculate travel time + parking |
| Event time changes | Recalculate travel time (traffic changes by time of day) |
| New attendee added | Check if internal/external, update classification |
| Conflict detected | Recommend resolution using scheduling rules |

### What {{AGENT_NAME}} Says When It Doesn't Know

```
"📅 New event: 'Meeting with TBD' — Wednesday 3:00 PM
 📍 No location specified

 I'm not sure about this one:
 ⚠ No location — is this virtual or in-person?
 ⚠ No attendees listed — can't classify internal/external
 ⚠ Can't calculate travel time without a location

 {{EA_NAME}}, can you fill in the details?
 [Add Location] [It's Virtual — Add Meet Link] [I'll Handle It]"
```

{{AGENT_NAME}} asks instead of guessing. If it doesn't have enough info, it flags and waits.

## Dependencies

- `gog` CLI (built-in OpenClaw tool — already installed on {{AGENT_NAME}})
- `goplaces` CLI (built-in OpenClaw tool — needs install + API key)
- Google Maps Directions API (for travel time with traffic — needs API key)
- Notion People DB (to determine if attendees are internal vs. external)
- {{EA_NAME}} on Telegram (for approval flow)
- `config/scheduling-rules.json` (shared with Meeting Prep, approved by {{EA_NAME}})
- Notion Oversight Agent (for calendar write operations)

## Status

DRAFT — needs full implementation. High priority per {{EA_NAME}} meeting. This is the scheduling system {{EA_NAME}} has been doing manually.
