# Deploy Telegram Communication Strategy

> **📌 REFERENCE SNAPSHOT — NOT FOR DEFAULT DEPLOYMENT.** This file was preserved from the {{PRINCIPAL_NAME}} / {{COMPANY_NAME}} engagement as a reference example of how a custom office/VC deployment was built. **Do NOT install this skill as part of a generic OpenClaw deployment.** It may contain engagement-specific defaults (emails, chat IDs, team names, VC-flavored workflows), Claude Opus routing (we default to Codex per `audit/_baseline.md` §BB in the parent repo), or stale "Blocked Commands" claims from 2026.2.26 that were resolved in 2026.4.15+. Use as inspiration when building custom-office skills, not as a default.

## Compatibility
- **OpenClaw Version**: 2026.3.27+
- **Status**: APPROVED
- **Layer**: 0 (Foundation) — deployed FIRST, before any skill that sends messages
- **Architecture**: Two Telegram supergroups. {{AGENT_NAME}} is admin. requireMentions: false always. Topics created via Bot API with welcome/example messages.
- **Approved**: 2026-03-30 by the operator, with {{EA_NAME}}'s input on group structure and urgent email routing.

## Purpose

Deploy the messaging infrastructure for all {{AGENT_NAME}} skills. Two groups: {{AGENT_NAME}} HQ ({{PRINCIPAL_NAME}}'s project-based group) and {{EA_NAME}} {{AGENT_NAME}} HQ ({{EA_NAME}}'s operations group). Every topic gets a welcome message with examples explaining what it's for, how to use it, and what to expect.

{{EA_NAME}} filters urgent emails before they reach {{PRINCIPAL_NAME}}. {{PRINCIPAL_NAME}}'s group is clean and project-focused. {{EA_NAME}}'s group handles all operational approvals. requireMentions: false everywhere — just type naturally.

## Topic Welcome Messages

When each topic is created, {{AGENT_NAME}} posts a pinned welcome message explaining the topic. These serve as onboarding — when {{PRINCIPAL_NAME}} or {{EA_NAME}} open a topic for the first time, they immediately understand what it's for.

### {{AGENT_NAME}} HQ ({{PRINCIPAL_NAME}}'s Group) — Welcome Messages

#### #edge (General)
```
👋 Welcome to {{AGENT_NAME}}

This is your main chat with me. Just type naturally — no @mentions needed.

Things you can say here:
• "Book dinner at Frasca for 2 at 7 PM Friday"
• "What's John Smith's email?"
• "Move my 3 PM to tomorrow"
• "What's on my calendar Thursday?"
• "Hey {{AGENT_NAME}}, make a new project for the Foundation website"
• "Research that fintech company from Austin"

I'll figure out the right skill and respond. If I need clarification, I'll ask.

For project-specific work, use the project topics below.
For personal stuff (dinner, golf, travel), use #personal.
```

#### #principal-briefing
```
📋 {{PRINCIPAL_NAME}}'s Daily Briefing

Every morning at 5:30 AM and every evening at 7 PM, you'll get a concise summary here.

Morning brief includes:
• Today's calendar with attendee context (who they are, why you're meeting)
• Items that need your attention today
• Overdue follow-ups
• Urgent email count

Evening summary includes:
• What was accomplished today
• Meetings attended
• New action items created
• Tomorrow's preview

Target: under 2,000 characters. Scannable in 60 seconds.

Example:
"Good morning, {{PRINCIPAL_NAME}}. Here's your Tuesday brief.

CALENDAR (3 meetings)
• 9:00 AM — John Smith, Sequoia (Series B for PortCo X)
  Last spoke Mar 15. Interested in Sundance.
• 2:00 PM — Founder intro via {{TEAM_ANALYST}} (research attached)
• 4:00 PM — {{TEAM_MEMBER_2}} sync (moveable)

NEEDS ATTENTION
• LP capital call response overdue (3 days)

3 urgent emails ({{EA_NAME}} reviewing). 12 total unread."
```

#### #principal-meetings
```
📅 Meeting Prep

Before your meetings, you'll see dossiers here with everything you need to know.

Each dossier includes:
• Who you're meeting (title, company, relationship history)
• WHY you're meeting (email thread that led to this)
• Their interests (Sundance, politics, music — for conversation)
• Travel time + parking + GPS link (for in-person meetings)
• Suggested talking points

Only external attendees get full research. Internal team is excluded.

Example:
"9:00 AM — Series B Discussion (📹 Zoom)
WHY: Inbound from Sequoia. {{TEAM_ANALYST}} intro'd Mar 15.
John Smith (MD, Sequoia) — 12 meetings since 2024
Interests: Sundance, politics
Talking points: Series B terms, PortCo X traction"

You can also say: "Prep my next meeting" anytime in #edge.
```

#### #principal-approvals
```
✅ {{PRINCIPAL_NAME}}'s Approvals

Items that only you can decide show up here. Tap a button to approve or reject.

Types of approvals:
• Investment decisions (deal terms, commitments)
• Major commitments (partnerships, sponsorships)
• Items {{EA_NAME}} forwards from urgent email triage

Example:
"📋 Investment Decision
PortCo X Series B — Sequoia proposing $50M pre-money.
{{TEAM_ANALYST}}'s analysis attached. Board meeting Thursday.
[Approve] [Reject] [Need More Info]"

You won't see operational approvals here — {{EA_NAME}} handles those in her group.
```

#### #boulder-roots (example project topic)
```
🎵 Boulder Roots Music Fest

Everything for BRMF lives here: planning, sponsor outreach, website development, event logistics.

What you'll see:
• 📋 TASK — Agent team breaking down your request
• ⚙️ IN PROGRESS — Builders working on it
• 🔍 REVIEW — Code review results
• 🔀 PR — Preview link + approve/merge button
• 🚀 DEPLOYED — Live on the website

Things you can say here:
• "Add sponsor tiers to the homepage — gold, silver, bronze"
• "Update the event schedule for June dates"
• "Make the mobile version look better"

I'll break it into tasks, build it overnight, and have a preview ready by morning.

[View Current Goals] [Add New Goal]
```

#### #investments
```
💰 Investments

Active deal flow, term sheets, due diligence tracking.

What you'll see:
• Deal updates and status changes
• Due diligence research from Nightman
• Meeting prep for investor meetings (cross-posted from #principal-meetings when relevant)
• Action items related to active deals

Things you can say:
• "What's the status on the PortCo X deal?"
• "Research this new company that pitched us"
• "Draft a follow-up to the Series A conversation"
```

#### #portfolio
```
📊 Portfolio

30+ portfolio company monitoring, LP reporting, dashboard updates.

What you'll see:
• Portfolio company updates and news
• Quarterly reporting status
• Dashboard improvements (dev work)
• LP communication drafts

Things you can say:
• "How is PortCo X performing this quarter?"
• "Draft the LP quarterly update"
• "Add a new chart to the dashboard showing ARR growth"
```

#### #contacts-vip
```
👥 Contacts & VIP Lists

Your persons directory and VIP management.

What you'll see:
• Notable contact updates (job changes, new ventures)
• VIP list curation for events
• Contact research results you requested

Things you can say:
• "Who do we know at Andreessen?"
• "Build a VIP list for the next Sundance reception"
• "Find me the fintech guy from Austin"
• "What's {{EXTERNAL_CONTACT_1}}'s preferred email?"
```

#### #ensuring-colorado
```
🏔️ Ensuring Colorado's Innovation Future

Policy, events, stakeholder management for this initiative.

What you'll see:
• Project updates and milestones
• Stakeholder research
• Event planning
• Website/content development

Things you can say:
• "Draft a stakeholder update email"
• "Research Colorado tech policy developments"
• "Build a landing page for the initiative"
```

#### #creators-edge
```
🎨 Creators at the {{AGENT_NAME}}

Community building, directory management, events.

What you'll see:
• Community updates
• Creator directory changes
• Event planning and coordination
• Content and outreach

Things you can say:
• "Add a new creator to the directory"
• "Plan the next community event"
• "Update the Creators at the {{AGENT_NAME}} website"
```

#### #foundation
```
🏛️ {{FAMILY_FOUNDATION}}

Grant tracking, nonprofit operations, quarterly board decks.

What you'll see:
• Grant applications and tracking
• Board deck drafts (narrative format)
• Foundation event planning
• Donor communications

Things you can say:
• "What grants are pending review?"
• "Draft the Q2 board deck"
• "Track this new grant application from [org]"
```

#### #bear-roars
```
🎙️ The Bear Roars Podcast

Guest coordination, episode scheduling, show notes.

What you'll see:
• Upcoming guest research and dossiers
• Episode scheduling and logistics
• Show notes drafts
• Listener engagement ideas

Things you can say:
• "Research [person] as a potential guest"
• "Schedule a recording for next week"
• "Draft show notes for the latest episode"
```

#### #personal
```
❤️ Personal

Dinner reservations, golf, Beaver Creek/Cabo houses, family calendar, travel, NetJets.

What you'll see:
• Reservation confirmations ("Frasca booked, 7 PM Friday")
• Travel logistics (NetJets hours, Fly1200 bookings)
• House reservation management
• Golf tournament registrations
• Family calendar updates

Things you can say:
• "Book dinner at Frasca for 2 at 7 PM Friday"
• "Check availability at the Beaver Creek house for July 4th weekend"
• "Register me for the Arrowhead golf tournament"
• "Book a NetJets flight to Cabo for Thursday"
• "What's on the family calendar this weekend?"

I can make calls in your voice for restaurant reservations. 🔊
```

#### #nightman-report
```
🌙 Nightman Report

{{AGENT_NAME}}'s overnight workforce runs from 10 PM to 5 AM. This is where you see what happened.

What you'll see:
• 6:00 PM — Tonight's plan: what will be worked on, what's missing
  [Approve Plan] [Add Tasks] [Modify]
• 5:00 AM — Night summary: what was accomplished, PRs created, ideas generated

Example evening plan:
"🌙 Tonight's Shift:
1. Calendar enrichment (3 meetings tomorrow)
2. Contact research (2 unknown attendees)
3. Boulder Roots: mobile sponsor page improvements
4. Draft 2 overdue follow-up emails

💡 Strategic thinking: sponsor acquisition strategy

⚠ Missing: no goals for Foundation website
[Approve Plan] [Add Tasks]"

Example morning summary:
"🌙 Nightman Complete — 14 cycles, $3.42
✓ Calendar enriched, ✓ 2 contacts researched
✓ PR #48: Sponsor tiers (preview ready)
💡 Idea: Tiered sponsor layout converts 23% more inquiries
[View PR] [View Idea]"
```

### {{EA_NAME}} {{AGENT_NAME}} HQ — Welcome Messages

#### #edge ({{EA_NAME}}'s General)
```
👋 Welcome to {{AGENT_NAME}} — {{EA_NAME}}'s workspace

This is your main chat with me. Just type naturally.

Things you can say:
• "Research {{EXTERNAL_CONTACT_1}} for {{PRINCIPAL_NAME}}'s meeting tomorrow"
• "What's on {{PRINCIPAL_NAME}}'s calendar Thursday?"
• "Draft a follow-up to the LP from last week"
• "Check if John Smith's contact info is up to date"
• "What emails came in overnight?"

I handle the request and respond here. Automated outputs go to the specific topics below.
```

#### #ea-briefing
```
📋 {{EA_NAME}}'s Operational Briefing

Every morning at 5:30 AM and evening at 7 PM, you'll get the full operational picture.

Morning brief includes:
• Calendar integrity report (missing links, travel time gaps)
• Scheduling intelligence (moveable vs. fixed meetings, conflicts)
• Draft follow-up emails awaiting your review
• Contact updates pending approval
• Full email digest (all urgency tiers)
• Action items (complete list with priorities)

This is more detailed than {{PRINCIPAL_NAME}}'s version (~6,000 chars vs his ~2,000). You see everything. He sees need-to-know only.

Tip: Review #ea-contacts (enrichment batch) BEFORE reading this briefing — the briefing references enrichment data.
```

#### #ea-urgent
```
🔴 Urgent Email Triage

ALL emails {{AGENT_NAME}} classifies as urgent (score 70+) come here FIRST. You decide what reaches {{PRINCIPAL_NAME}}.

For each urgent email, you'll see:
• Sender and subject
• Urgency score + why it's urgent
• The actual ASK (not just "urgent from Sequoia" but "John wants the metrics you promised")
• Thread summary (full conversation context)
• Relationship context from Knowledge Graph

Your options:
[Forward to {{PRINCIPAL_NAME}}] — appears in his #principal-approvals
[Handle It] — you respond directly
[Dismiss] — logged but not forwarded

Example:
"🔴 URGENT — Score: 87
From: John Smith (MD, Sequoia)
Subject: Re: PortCo X Metrics
THE ASK: Send remaining metrics. Committee Tuesday.
CONTEXT: {{PRINCIPAL_NAME}} promised Mar 28. 7-message thread over 3 weeks.
[Forward to {{PRINCIPAL_NAME}}] [Handle It] [Dismiss]"

{{PRINCIPAL_NAME}} never sees false positives. You're the filter.
```

#### #ea-scheduling
```
📅 Calendar Heartbeat

Every 30 minutes, I scan {{PRINCIPAL_NAME}}'s calendar for changes. New or modified events show up here for your review.

What you'll see:
• New events {{EA_NAME}} or {{PRINCIPAL_NAME}} added → I suggest enrichment (travel time, parking, links)
• Missing video links on virtual meetings → I flag them
• Scheduling conflicts → I recommend which meeting to move (based on your ranking rules)
• Travel time calculations with GPS directions

Example:
"📅 New event detected:
'Lunch with {{EXTERNAL_CONTACT_1}}' — Tuesday 12:00 PM
📍 Frasca Food & Wine, Boulder

I can add:
✓ Travel time: 15 min from office
✓ Parking: Street parking on Pine St
✓ Return travel: 15 min block
[Approve All] [Modify] [Skip]"

If I don't have enough info, I'll ask instead of guessing.
```

#### #ea-drafts
```
✉️ Email Drafts

When {{AGENT_NAME}} drafts emails in {{PRINCIPAL_NAME}}'s voice, they show up here for your review before sending.

Sources:
• Nightman overnight draft follow-ups (overdue items)
• {{PRINCIPAL_NAME}}'s requests ("draft a reply to John")
• Action item follow-ups

For each draft you'll see:
• Who it's to (preferred email + EA CC if applicable)
• The draft text in {{PRINCIPAL_NAME}}'s voice
• Why it was drafted (context)

Your options:
[Approve — Send] [Edit] [Reject]

Hyperlinks are always included. Formatting is always full-width. EA is always CC'd when known. These are {{PRINCIPAL_NAME}}'s rules from your meeting.
```

#### #ea-contacts
```
👥 Contact Enrichment

Every morning at 5:00 AM (30 min before your briefing), overnight enrichment results land here.

Grouped by confidence:
• HIGH (90%+): title changes, new tags — likely correct
  [Approve All High] [Review Each]
• MEDIUM (70-89%): worth a look
  [Approve] [Reject] per item
• NEEDS INPUT (<50%): I'm not sure
  [Answer] or [Skip]

Example:
"HIGH CONFIDENCE (5 items):
✓ John Smith: title Partner → MD (LinkedIn, 0.94)
✓ {{EXTERNAL_CONTACT_1}}: added 'climate tech' interest (transcript, 0.91)
[Approve All High-Confidence]

NEEDS INPUT (1 item):
❓ 'J. Smith' — same as John Smith? 0.45 confident
[Yes — Merge] [No — Keep Separate]"

This system learns. Eventually high-confidence items will auto-approve (you control when).
```

#### #ea-actions
```
📋 Action Items & Approvals

After meetings are processed, action items show up here. Also: Notion write approvals and operational tasks.

What you'll see:
• Extracted action items with owners (who should do this?)
• Items {{AGENT_NAME}} is confident about: [Approve] [Edit] [Reject]
• Items {{AGENT_NAME}} is NOT confident about: ownership unclear, needs your input
• Notion write approval requests (from oversight agent)

Example:
"📋 3 items from 'Series B Discussion':

1/3: Send PortCo X metrics to John
  Owner: {{PRINCIPAL_NAME}} ✓ (confidence 0.95)
  Due: Fri Apr 4 | Priority: High
  [Approve] [Edit] [Reject]

2/3: Prepare board deck scenarios
  Owner: {{TEAM_ANALYST}} ✓ (confidence 0.92)
  [Approve] [Edit] [Reject]

3/3: Pull comparative market data
  Owner: ❓ UNCLEAR (confidence 0.30)
  {{TEAM_ANALYST}} or Gibson could do this.
  [Assign {{TEAM_ANALYST}}] [Assign Gibson] [Assign {{PRINCIPAL_NAME}}]"
```

#### #security-council
```
🛡️ Security Council

Every night at 3:30 AM, four AI security personas audit {{AGENT_NAME}}'s system. Results posted here.

Personas:
1. Offensive Security — thinks like an attacker
2. Defensive Security — checks protection controls
3. Data Privacy — hunts for PII exposure
4. Operational Realism — filters false positives

You'll see: severity counts, new findings, and critical alerts (critical alerts are IMMEDIATE — not batched).

Example:
"🛡️ Security Council Report #47
🔴 Critical: 0 | 🟠 High: 1 | 🟡 Medium: 3 | 🔵 Low: 2
New findings: 2 | Dismissed: 4

🟠 HIGH: npm package 'lodash' has CVE-2026-1234
  Recommendation: Update to 4.17.22
  [View Details]"
```

#### #system-health
```
⚙️ System Health

Errors, failures, and operational status for {{AGENT_NAME}}'s infrastructure.

What posts here:
• LaunchAgent failures (any scheduled task that crashes)
• Git sync conflicts
• Backup completion/failure
• Knowledge Graph errors (Docker, Neo4j)
• Transcript pipeline status
• API connectivity issues
• Disk space warnings

This is the "something went wrong" channel. If everything is working, it's quiet.

Example:
"⚠️ LaunchAgent failure: com.edge.email-intel
Exit code: 1 | Error: ANTHROPIC_API_KEY invalid
Last success: 2 hours ago
[View Logs] [Restart]"
```

#### #nightman-ea
```
🌙 Nightman — {{EA_NAME}}'s View

Your view of what Nightman worked on overnight, specifically items that need your action.

What you'll see:
• Draft follow-up emails awaiting your review (overdue items)
• Contact enrichment results pending approval
• Calendar enrichment applied (travel time, parking added)
• Items that are blocked and need your input

This arrives ~5:00 AM, 30 min before your briefing.

Example:
"🌙 Items for your review:

DRAFTS (3 follow-ups):
1. LP capital call — {{PRINCIPAL_NAME}} hasn't responded in 3 days [Review Draft]
2. Sarah @ Foundation — Sundance invite [Review Draft]
3. Vendor invoice — needs signature [Review Draft]

CALENDAR:
✓ Added travel time to 2:00 PM meeting (25 min)
✓ All virtual meetings have Meet links

CONTACTS:
5 enrichments pending your review in #ea-contacts"
```

### {{EA_NAME}}'s Forwarding to {{PRINCIPAL_NAME}}

When {{EA_NAME}} taps [Forward to {{PRINCIPAL_NAME}}] on any item, {{AGENT_NAME}}:
1. Reformats for {{PRINCIPAL_NAME}}'s concise style (strips operational details)
2. Posts to the appropriate {{PRINCIPAL_NAME}} topic (#principal-approvals or project topic)
3. Adds {{EA_NAME}}'s note if she typed one
4. Logs the forwarding action

Example — {{EA_NAME}} forwards an urgent email:
```
In #ea-urgent, {{EA_NAME}} taps [Forward to {{PRINCIPAL_NAME}}] on John's email.

In {{PRINCIPAL_NAME}}'s #principal-approvals:
"📧 Forwarded by {{EA_NAME}}:
From: John Smith (Sequoia)
{{EA_NAME}}'s note: "This is real — committee is Tuesday. He needs the metrics."
THE ASK: Send remaining PortCo X metrics
[Reply] [Remind Me Tomorrow] [{{EA_NAME}} — Handle It]"
```

## What Gets Installed

| Component | Path | Purpose |
|-----------|------|---------|
| Group setup guide | `docs/telegram-setup-guide.md` | Create both groups, add {{AGENT_NAME}} as admin |
| Topic creation + welcome messages | `scripts/telegram-setup-topics.js` | Creates all topics, posts + pins welcome messages |
| Welcome message templates | `config/telegram-welcome-messages.json` | All welcome messages above, templated for variable substitution |
| Project creator | `scripts/telegram-new-project.js` | "Hey {{AGENT_NAME}}, new project" → topic + welcome message + agent team |
| Topic registry | `config/telegram-topics.json` | All topic thread IDs |
| Routing config | `config/telegram-notifications.json` | Skill → topic mapping |
| Forwarding handler | `scripts/telegram-forward-to-principal.js` | {{EA_NAME}} taps [Forward to {{PRINCIPAL_NAME}}] → reformats + posts to {{PRINCIPAL_NAME}}'s topic |
| Reboot check | Nightman system health | Verify requireMentions=false every cycle |
| TOOLS.md | `TOOLS.md` | create_project, list_topics, post_to_topic, forward_to_dan |

## Cost

$0/month.

## Testing

| Phase | What | Target |
|-------|------|--------|
| T1 | Create both supergroups | Telegram |
| T2 | {{AGENT_NAME}} bot admin in both | Telegram |
| T3 | Run topic creation → verify all topics + welcome messages pinned | Both groups |
| T4 | Read every welcome message → verify examples are accurate and helpful | Manual review |
| T5 | Post test message to every topic → verify delivery | Both groups |
| T6 | Inline keyboard → tap approve → verify callback | Test topic |
| T7 | Urgent email → #ea-urgent → {{EA_NAME}} taps [Forward to {{PRINCIPAL_NAME}}] → verify appears in #principal-approvals | Both groups |
| T8 | New project creation → topic + welcome message | {{PRINCIPAL_NAME}}'s group |
| T9 | requireMentions: false → type without @{{AGENT_NAME}} → {{AGENT_NAME}} responds | Both groups |
| T10 | Full routing audit: every skill → correct topic | Cross-check |
| T11 | {{PRINCIPAL_NAME}} notification check | {{PRINCIPAL_NAME}}'s phone |
| T12 | {{EA_NAME}} notification check | {{EA_NAME}}'s phone |

## Dependencies

Deploys FIRST. Every messaging skill depends on this.
