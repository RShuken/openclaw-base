# USER (template)

Who the operator is. Loaded every turn as context so the agent knows who it's serving. This file is about the PERSON, not the agent (that's IDENTITY.md).

Replace every `{{...}}` before deploying. Target: 30-60 lines.

---

## Identity

- **Name:** {{OWNER_NAME}}
- **Preferred name / nickname:** {{NICKNAME or "same as name"}}
- **Pronouns:** {{PRONOUNS}}
- **Email:** {{EMAIL}}
- **Timezone:** {{TIMEZONE}}

## Role and context

- **Primary role:** {{e.g. "Managing Director at {{COMPANY_NAME}}" / "Independent OpenClaw consultant" / "Software engineer at X"}}
- **Company / org:** {{ORG}}
- **What they're building right now:** {{high-level — 1-2 sentences}}

## How they work

- **Working hours:** {{e.g. "9am-7pm PT weekdays; light weekend availability"}}
- **Communication preferences:** {{e.g. "prefers Telegram over email for quick questions; async > sync"}}
- **Decision-making style:** {{e.g. "asks for the range of options + a recommendation; doesn't want a single answer presented as the only path"}}
- **Time to respond:** {{e.g. "30 min for urgent; 24h for low-priority"}}

## What they know

- **Technical depth:** {{e.g. "deep backend + infra; light on frontend" / "non-technical, needs plain-language explanations"}}
- **Familiar with:** {{list of technologies / domains}}
- **Learning right now:** {{optional — what they're actively learning, so the agent can be pedagogical when appropriate}}

## What they care about

- {{VALUE_1 — e.g. "Privacy. Data stays on-device when possible."}}
- {{VALUE_2 — e.g. "Low ongoing cost. Prefer local models over frontier cloud unless quality genuinely matters."}}
- {{VALUE_3 — e.g. "Moving fast. Prefer shipping a rough thing + iterating over shipping nothing."}}

## Hard nos

- {{NO_1 — e.g. "Do not send any email to anyone without my explicit OK"}}
- {{NO_2 — e.g. "Do not speculate about my health, finances, or family"}}
- {{NO_3 — e.g. "Do not claim you did something you didn't"}}

## Context for the agent

{{Optional — 2-3 sentences the agent should always keep in mind about the owner's situation. E.g. "Running a solo consulting business. Time is the constraint, not money. Prefers to decide fast and iterate rather than plan exhaustively."}}

---

## How this file is different from MEMORY.md

- **USER.md** = stable facts about who the owner is. Changes rarely (promotion, move, value shift). Short.
- **MEMORY.md** = active current-state index (this week's focus, current projects, standing preferences). Changes weekly or so.
- If you're unsure where something goes: if it's true about this person for years, USER.md. If it's true this quarter, MEMORY.md.
