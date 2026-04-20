# Deploy Contact Auto-Enrichment

> **📌 REFERENCE SNAPSHOT — NOT FOR DEFAULT DEPLOYMENT.** This file was preserved from the {{PRINCIPAL_NAME}} / {{COMPANY_NAME}} engagement as a reference example of how a custom office/VC deployment was built. **Do NOT install this skill as part of a generic OpenClaw deployment.** It may contain engagement-specific defaults (emails, chat IDs, team names, VC-flavored workflows), Claude Opus routing (we default to Codex per `audit/_baseline.md` §BB in the parent repo), or stale "Blocked Commands" claims from 2026.2.26 that were resolved in 2026.4.15+. Use as inspiration when building custom-office skills, not as a default.

## Compatibility
- **OpenClaw Version**: 2026.3.27+
- **Status**: DRAFT — needs full research and implementation
- **Layer**: 2 (Notion + CRM) — enriches the People and Companies databases
- **Architecture**: Standalone Node.js + Tavily/web search + Notion API

## Purpose

Install an automated contact enrichment engine that replaces {{EA_NAME}}'s manual process of researching contacts on LinkedIn, tagging their interests, updating their titles/companies, and maintaining the People directory after every meeting.

**Why this exists:** {{EA_NAME}}'s top time-wasting task (2026-03-30 meeting). Every time {{PRINCIPAL_NAME}} meets someone, {{EA_NAME}} manually: (1) checks if they're in Notion, (2) looks them up on LinkedIn, (3) tags them as investor/tech entrepreneur/etc., (4) adds interest tags (Sundance, music, politics), (5) updates their title and company if changed, (6) adds meeting notes. She described it as "a dummy could do it, but it takes time." This is exactly what an AI agent should automate.

**Why it's needed by other skills:**
- **Daily Briefing** accuracy agent verifies contact data — enrichment keeps it current
- **Meeting Prep** builds attendee dossiers from enriched contact records
- **Personal CRM** is the local cache — enrichment updates the source (Notion) which syncs to CRM
- **Nightman** runs enrichment overnight for tomorrow's meeting attendees
- **Action Items** matches names from transcripts against enriched contact records
- **Email drafting** needs preferred emails and EA CC rules — stored in enriched contact records

**What enrichment does per contact:**
1. **LinkedIn lookup** (via Tavily web search): Find current title, company, bio, connections
2. **Interest tagging**: Analyze LinkedIn bio, recent activity, and meeting transcript mentions for interests (Sundance, music, politics, AI, crypto, etc.)
3. **Company enrichment**: If person has a company, look up: industry, stage, funding, relationship to {{COMPANY_NAME}}
4. **Preferred email selection**: If multiple emails known, mark the primary business email as preferred
5. **EA identification**: If the contact has an EA (mentioned in emails or calendar invites), add EA name and email
6. **Staleness detection**: Flag records not updated in >90 days for re-enrichment
7. **Write-back**: Update Notion People DB with enriched data. Update local SQLite CRM cache.

**Enrichment triggers:**
- **After every meeting**: Transcript pipeline extracts attendees → enrichment runs for any attendee with incomplete records
- **Nightly (Nightman)**: Enrich all attendees for tomorrow's calendar
- **On demand**: {{EA_NAME}} or {{PRINCIPAL_NAME}} requests enrichment for a specific contact
- **Bulk**: Initial enrichment of the 20K+ People directory (batched over multiple nights)

## Enrichment Safety Rules (Non-Negotiable)

These rules are HARD — no override, no exception, no "just this once":

### Data Protection

| Rule | Why |
|------|-----|
| **NEVER delete a contact record** | Deleting loses all history. If a contact is wrong, flag it — don't delete it. |
| **NEVER delete a field value** | If CRM says "Partner" and new data says nothing, KEEP "Partner". Never overwrite with empty. |
| **NEVER remove interest tags** | Tags are additive only. If a tag was wrong, {{EA_NAME}} removes it manually via Notion. {{AGENT_NAME}} only adds. |
| **NEVER overwrite with less data** | If current record has 15 fields populated and enrichment only found 8, keep all 15. Only update the 8 that have newer/better data. |
| **NEVER merge contacts without human approval** | "Is this the same John Smith?" → ask {{EA_NAME}}, don't auto-merge. |
| **NEVER modify records for internal team** | CVT team members are excluded from enrichment entirely. |

### Change Tracking

| Rule | Why |
|------|-----|
| **Every change is logged with before + after** | If {{AGENT_NAME}} changes title from "Partner" to "MD", the log shows both values + source + timestamp |
| **Every change has a source citation** | "Title updated to MD (source: Tavily LinkedIn search, 2026-03-31)" — never unsourced changes |
| **Every change goes through Notion Oversight Agent** | Sonnet reviews the change for safety → {{EA_NAME}} approves via Telegram |
| **All enrichment batched for morning approval** | Nothing is written to CRM/Notion until {{EA_NAME}} reviews. ALL overnight and post-meeting enrichment queues up and gets sent to {{EA_NAME}} in the `#contact-enrichment` Telegram topic each morning (with her 5:30 AM briefing). She approves, modifies, or rejects each batch. No enrichment is applied without her sign-off. |

### What Happens When Data Conflicts

| Situation | What {{AGENT_NAME}} Does |
|-----------|---------------|
| CRM says "Partner", Tavily says "MD" | Flag: "Possible title change: Partner → MD. Source: LinkedIn (Mar 31). [Approve Update] [Keep Current] [Flag for {{EA_NAME}}]" |
| CRM says "Sequoia", email domain is @newco.com | Flag: "Possible job change: Sequoia → NewCo. Email from john@newco.com on Mar 31. [Confirm Change] [Keep Current]" |
| Two CRM records might be same person | Flag: "Possible duplicate: 'John Smith (john@sequoia.com)' and 'J. Smith (jsmith@sequoia.com)'. [Merge — {{EA_NAME}} Reviews] [Keep Separate]" |
| Tavily returns different data than Notion | Notion wins (source of truth) UNLESS the Tavily data is clearly newer (e.g., new job title on LinkedIn). Flag for {{EA_NAME}} either way. |
| Enrichment finds no data | Log "no results found" — never create a blank record or clear existing data |

### Double-Check for Destructive Operations

Even with all the above rules, the Notion Oversight Agent provides a SECOND safety layer:

```
Contact Enrichment wants to update John Smith's title:
  Before: "Partner"
  After: "Managing Director"
  Source: Tavily LinkedIn search, 2026-03-31
         ↓
Oversight Agent reviews:
  ✓ Not a deletion (updating, not removing)
  ✓ Not blanking a field (new value is populated)
  ✓ Source is cited
  ✓ Change makes sense (title changes happen)
  → Verdict: SAFE
         ↓
{{EA_NAME}} receives Telegram:
  "{{AGENT_NAME}} wants to update John Smith:
   Title: Partner → Managing Director
   Source: LinkedIn (Mar 31)
   Oversight: ✓ Safe
   [Approve] [Reject]"
         ↓
{{EA_NAME}} approves → CRM + Notion updated
{{EA_NAME}} rejects → change discarded, logged
```

**Two independent safety checks (enrichment rules + oversight agent) PLUS {{EA_NAME}}'s human approval. Three layers before any data changes.**

## Dependencies

- Notion API (read/write People + Companies databases)
- Tavily API (web search for LinkedIn research)
- Personal CRM SQLite (local cache updates)
- Vector embeddings (for semantic matching — "is John Smith in the CRM the same John Smith in this email?")
- Knowledge Graph (for temporal relationship tracking — job changes over time)
- Notion Oversight Agent (REQUIRED — all writes go through safety review)
- {{EA_NAME}} on Telegram (REQUIRED — all changes need human approval)

## Confidence-Based Trust System (Phased Automation)

Every enrichment action gets a confidence score (0-1.0) and a reason. Over time, the system learns which actions {{EA_NAME}} always approves and which need human judgment.

### Confidence Scoring Per Enrichment

```json
{
  "contact": "John Smith",
  "action": "update_title",
  "before": "Partner",
  "after": "Managing Director",
  "confidence": 0.92,
  "reason": "LinkedIn profile clearly states 'Managing Director' as of Jan 2026. Multiple recent articles confirm.",
  "recommendation": "auto_approve",
  "source": "Tavily LinkedIn search, 2026-03-31"
}
```

| Confidence | Recommendation | What It Means |
|-----------|---------------|--------------|
| **0.9 - 1.0** | `auto_approve` | High confidence. Source is clear, data is unambiguous. "LinkedIn says MD, three sources confirm." |
| **0.7 - 0.89** | `approve_with_context` | Likely correct but worth a glance. "Tavily found new company, but only one source." |
| **0.5 - 0.69** | `needs_review` | Uncertain. "Title might have changed but search results are mixed." |
| **0.0 - 0.49** | `needs_human_input` | Don't know enough. "Possible duplicate contact. Can't determine if same person." |

### {{EA_NAME}}'s Morning Enrichment Report

Each morning in `#contact-enrichment`, {{EA_NAME}} sees enrichments grouped by confidence:

```
📋 Contact Enrichment Report — March 31

HIGH CONFIDENCE (5 items) — I'm 90%+ sure these are right:
  ✓ John Smith: title Partner → MD (LinkedIn confirmed, 0.94)
  ✓ {{EXTERNAL_CONTACT_1}}: added interest "climate tech" (mentioned in Mar 30 transcript, 0.91)
  ✓ Alex Rivera: new contact created from tomorrow's calendar (Tavily research, 0.93)
  ✓ Marcus Lee: company updated to FinStack (LinkedIn + email domain match, 0.95)
  ✓ Jane Doe: added as EA for John Smith (CC'd on 4/5 emails, 0.90)
  [Approve All High-Confidence] [Review Each]

MEDIUM CONFIDENCE (2 items) — Worth a look:
  ⚠ Tom Park: possible job change, Sequoia → a16z (email from new domain, 0.78)
  ⚠ Lisa Wang: added interest "crypto" (mentioned once in passing, 0.72)
  [Approve] [Reject] [Modify] for each

NEEDS YOUR INPUT (1 item) — I'm not sure:
  ❓ "J. Smith (jsmith@sequoia.com)" — is this the same John Smith we already have?
     I'm 0.45 confident. Different email format, same company.
  [Yes — Merge] [No — Keep Separate] [I'll Check]
```

### Learning From {{EA_NAME}}'s Decisions

| {{EA_NAME}}'s Action | What the System Learns |
|---------------|----------------------|
| Approves a high-confidence item | Reinforces: this type of enrichment at this confidence level is safe |
| Rejects a high-confidence item | RED FLAG: confidence scoring was wrong. Log why. Adjust scoring model. |
| Approves a medium-confidence item | This category can be promoted toward auto-approve over time |
| Rejects a medium-confidence item | Correctly flagged as uncertain. Keep requiring review. |
| Answers a "needs input" question | Ground truth added. Future similar cases get higher confidence. |
| Batch-approves all high-confidence | Strong signal: these enrichment types are trusted |

### Trust Levels ({{EA_NAME}}/{{PRINCIPAL_NAME}} Toggle)

| Level | What Happens | When to Use |
|-------|-------------|-------------|
| **Level 1: Full Review** (default) | ALL enrichments batched for {{EA_NAME}}'s morning approval. Nothing auto-applied. | Day 1. No trust established yet. |
| **Level 2: Auto-Apply High Confidence** | Items scoring ≥ 0.9 auto-applied to CRM (still visible in morning report for spot-checking). Medium + low still need approval. | After 2-4 weeks, if {{EA_NAME}}'s approval rate for high-confidence items is >95%. |
| **Level 3: Auto-Apply Medium+** | Items scoring ≥ 0.7 auto-applied. Only low-confidence and "needs input" items require {{EA_NAME}}. | After 1-2 months, if approval rate for medium items is >90%. |
| **Level 4: Exception Only** | Everything auto-applied. {{EA_NAME}} only sees items that need human input (<0.5 confidence) or conflicts. | When the system has proven itself over months. |

**{{EA_NAME}} and {{PRINCIPAL_NAME}} decide when to advance levels.** {{AGENT_NAME}} tracks and reports its accuracy:

```
Weekly Enrichment Accuracy Report:
  Total enrichments this week: 47
  Auto-approved (high confidence): 32 — {{EA_NAME}} confirmed 31/32 (96.9%)
  Reviewed (medium): 11 — {{EA_NAME}} approved 9, rejected 2
  Needed input (low): 4 — {{EA_NAME}} resolved all 4

  Recommendation: Ready to advance to Level 2?
  [Yes — Enable Auto-Apply for High Confidence] [Not Yet]
```

### Config

```json
// config/enrichment.json
{
  "trustLevel": 1,
  "autoApplyThreshold": 0.9,
  "reviewThreshold": 0.7,
  "humanInputThreshold": 0.5,
  "trustLevelHistory": [
    { "level": 1, "setBy": "the operator", "setAt": "2026-03-31", "reason": "Initial deployment" }
  ],
  "weeklyAccuracy": [],
  "note": "{{EA_NAME}} and {{PRINCIPAL_NAME}} control trust level advancement. {{AGENT_NAME}} recommends but never auto-promotes."
}
```

## Status

DRAFT — needs full implementation. High priority per {{EA_NAME}} meeting. Starts at Level 1 (full review). Designed to earn trust and graduate to autonomous over time. {{EA_NAME}} controls the switch.
