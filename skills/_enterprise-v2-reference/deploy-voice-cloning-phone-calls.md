# Deploy Voice Cloning Phone Calls

> **📌 REFERENCE SNAPSHOT — NOT FOR DEFAULT DEPLOYMENT.** This file was preserved from the {{PRINCIPAL_NAME}} / {{COMPANY_NAME}} engagement as a reference example of how a custom office/VC deployment was built. **Do NOT install this skill as part of a generic OpenClaw deployment.** It may contain engagement-specific defaults (emails, chat IDs, team names, VC-flavored workflows), Claude Opus routing (we default to Codex per `audit/_baseline.md` §BB in the parent repo), or stale "Blocked Commands" claims from 2026.2.26 that were resolved in 2026.4.15+. Use as inspiration when building custom-office skills, not as a default.

## Compatibility
- **OpenClaw Version**: 2026.3.27+
- **Status**: APPROVED
- **Layer**: 1 (Communication) — killer feature
- **Architecture**: Vapi.ai (MCP server + telephony orchestration) + ElevenLabs (voice cloning) + Claude Opus (conversation logic) + Twilio (phone line) + Deepgram (speech-to-text)
- **Created**: 2026-03-30. Research: `docs/research/2026-03-30-voice-cloning-phone-calls.md`

## Purpose

Enable {{AGENT_NAME}} to make phone calls in {{PRINCIPAL_NAME}}'s cloned voice — restaurant reservations, travel bookings, venue inquiries. Vapi's MCP server connects directly to Claude/OpenClaw so {{PRINCIPAL_NAME}} or {{EA_NAME}} can say "book a table at Frasca for 7 PM Friday" and {{AGENT_NAME}} handles the entire call autonomously.

**Stack: Vapi + ElevenLabs + Claude (production approach, not MVP)**

## Setup

### Step 1: Clone {{PRINCIPAL_NAME}}'s Voice (One-Time, 15 min)
- Record 1-2 minutes of {{PRINCIPAL_NAME}} speaking naturally (or extract from podcast/meeting recording)
- Upload to ElevenLabs via API or dashboard → get voice_id
- {{PRINCIPAL_NAME}} must consent (ElevenLabs requires explicit permission)
- ElevenLabs Starter plan: $5/mo

### Step 2: Vapi Account + Phone Number
- Create Vapi account at vapi.ai
- Import or buy a Twilio phone number ($1.15/mo)
- Create a Vapi assistant:
  - LLM: Anthropic Claude (bring your own API key)
  - TTS: ElevenLabs with {{PRINCIPAL_NAME}}'s cloned voice_id
  - STT: Deepgram (best latency)
  - System prompt: conversation rules (polite, natural, {{PRINCIPAL_NAME}}'s style)

### Step 3: Install Vapi MCP Server
```json
{
  "mcpServers": {
    "vapi": {
      "command": "npx",
      "args": ["-y", "@vapi-ai/mcp-server"],
      "env": {
        "VAPI_TOKEN": "<vapi_token>"
      }
    }
  }
}
```

### Step 4: {{AGENT_NAME}} Can Make Calls
"Call Frasca and book a table for 2 at 7 PM Friday" → {{AGENT_NAME}} handles the entire call.

## Call Flow

```
{{PRINCIPAL_NAME}}/{{EA_NAME}} via Telegram: "Book a table at Frasca for 2 at 7 PM Friday"
         ↓
{{AGENT_NAME}} parses request + looks up phone number (goplaces)
         ↓
{{AGENT_NAME}} generates conversation script via Claude:
  "Request table for 2, 7 PM Friday April 4.
   If unavailable, try 6:30-8 PM range.
   If fully booked, ask about Saturday."
         ↓
{{EA_NAME}} approves: [Confirm Call] [Modify] [Cancel]
         ↓
Vapi places the call using {{PRINCIPAL_NAME}}'s cloned voice
  Claude handles real-time conversation with restaurant
         ↓
Call complete → result reported:
  "✓ Frasca, 2 people, 7 PM Friday. Table by the window.
   [Add to Calendar] [Send Confirmation to {{EA_NAME}}]"
```

## Approval Flow

| Call Type | Who Approves |
|-----------|-------------|
| Restaurant reservation | {{EA_NAME}} or auto if {{PRINCIPAL_NAME}} says "book it" |
| Travel booking (NetJets/Fly1200) | {{EA_NAME}} always |
| Venue inquiry (no commitment) | Auto — just asking questions |
| Any call making a commitment | {{EA_NAME}} or {{PRINCIPAL_NAME}} must approve |

## Safety Rules

1. {{PRINCIPAL_NAME}} must consent to voice cloning (legal requirement)
2. Never auto-call without human approval
3. Never call {{PRINCIPAL_NAME}}'s business contacts without explicit instruction
4. Log every call (transcript, duration, result in SQLite)
5. Call hours: 9 AM - 9 PM only
6. Restaurant/travel calls: no disclosure needed (personal, low risk)
7. Business calls: disclose "AI assistant calling on behalf of {{PRINCIPAL_NAME}}"

## What Gets Installed

| Component | Path | Purpose |
|-----------|------|---------|
| Vapi MCP server | OpenClaw MCP config | Claude triggers calls natively |
| Call handler | scripts/voice-call.js | Parse request, build script, initiate call |
| Call log DB | data/voice-calls.db | History: who, when, result, transcript |
| Config | config/voice-calling.json | ElevenLabs voice_id, Vapi token, Twilio number, rules |
| TOOLS.md | TOOLS.md | make_call, call_history, call_status |

## Cost

| Component | Cost |
|-----------|------|
| ElevenLabs Starter | $5/mo |
| Vapi pay-as-you-go | ~$2-5/mo |
| Twilio phone number | $1.15/mo |
| Claude API (conversation) | ~$1-3/mo |
| **Total** | **~$10-15/month** |

## Testing

| Phase | What | Target |
|-------|------|--------|
| T1 | #test-voice-calls topic | Test group |
| T2 | Clone the operator's voice (NOT {{PRINCIPAL_NAME}}'s) → verify quality | Local |
| T3 | Test call to the operator's phone → verify sounds natural | the operator's phone |
| T4 | Conversation test: "book a table for 2" → verify Claude handles back-and-forth | the operator plays host |
| T5 | Call logging → verify transcript in SQLite | Check DB |
| T6 | Calendar integration: confirmed reservation → event created | Test calendar |
| T7 | Approval flow: {{AGENT_NAME}} proposes → {{EA_NAME}} approves → call placed | Test topic |
| T8 | Safety: attempt call without approval → verify blocked | Check logs |
| T9 | Clone {{PRINCIPAL_NAME}}'s voice (with consent) → verify quality | {{PRINCIPAL_NAME}} listens |
| T10 | Live test: call a real restaurant | {{PRINCIPAL_NAME}} approves |
| T11 | Go-live | Monitor |

## Dependencies

- ElevenLabs account + {{PRINCIPAL_NAME}}'s voice consent
- Vapi account + Twilio phone number
- Claude API (conversation logic)
- goplaces (phone number lookup)
- Google Calendar API (auto-add reservations)
- Voice Engine ({{PRINCIPAL_NAME}}'s conversation tone)
