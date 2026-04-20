# MEMORY (template)

Curated long-term memory index. Loaded into system context **every turn** — so keep it tight.

> **🔒 Security gate:** MEMORY.md must NEVER be loaded in group chats or sub-agent sessions. The AGENTS.md boot sequence enforces this. Applies regardless of which memory backend is installed (see `skills/memory-options/community-patterns.md`).

Replace every `{{...}}` placeholder before deploying. Target: **< 200 lines** (the "Hot tier" per `skills/memory-options/community-patterns.md` tiering pattern). When this grows past 200 lines, promote older entries into `workspace/memory/archive/<year>/<week>.md` and leave only the currently-relevant index here.

---

## Owner profile

Quick facts the agent always needs about the operator.

- **Name:** {{OWNER_NAME}}
- **Timezone:** {{TIMEZONE}}
- **Primary role:** {{ROLE}}
- **Primary projects / orgs:** {{PROJECT_1}}, {{PROJECT_2}}
- **Communication style preferences:** {{e.g. "terse, no filler, no sycophancy" — cross-references SOUL.md}}

## Current focus

What the operator is actively working on right now. Update weekly.

- **This week's focus:** {{CURRENT_FOCUS}}
- **Active projects:** {{bulleted list}}
- **Blocked on:** {{if anything}}

## Key people

Top ~10 people the agent needs context on. Not exhaustive — the CRM / Graphiti is the full index. This is just the people the agent talks about most.

| Name | Role / relationship | Notes |
|------|---------------------|-------|
| {{NAME}} | {{ROLE}} | {{short context}} |

## Standing preferences

Things the operator has told the agent "always do X" or "never do Y." Survives across sessions.

- {{PREFERENCE_1}}
- {{PREFERENCE_2}}
- {{PREFERENCE_3}}

## Recurring patterns / decisions

Decisions made once that shouldn't be re-litigated. The agent cites these when a similar question comes up.

- {{DECISION_1 — e.g. "Use Supabase, not Firebase, for Cora"}}
- {{DECISION_2}}

## Dormant / archived (pointers only)

When entries age out of active focus, replace them with a pointer to where the detail lives:

- {{TOPIC}} — archived {{DATE}}, detail at `workspace/memory/archive/{{YEAR}}/{{WEEK}}.md`

---

## How this file grows

1. Heartbeat auto-save (`skills/memory-options/community-patterns.md`) appends to `workspace/memory/YYYY-MM-DD.md` — NOT to this file
2. Weekly on Monday, the rollup script promotes last week's daily notes into `workspace/memory/archive/YYYY/week-WW.md`
3. During rollup, the agent updates THIS file by adding any truly-long-term facts that emerged
4. When this file passes 200 lines, demote the least-used section into an archive file and leave a pointer

## Do not put here

- Transactional chat history (that's what `workspace/memory/YYYY-MM-DD.md` + the vector index are for)
- Conversation logs (same)
- One-off questions answered (same)
- Secrets / credentials (never in any workspace file — use `auth-profiles.json`)
- Group-chat-contaminable info (MEMORY.md is for private main-session context only)
