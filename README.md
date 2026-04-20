# openclaw-base

Clean, source-of-truth template library for installing and configuring OpenClaw on a fresh machine.

## What This Is

A publishable base that OpenClaw operators can fork for their own deployments (personal agents, client engagements, internal team bots). It contains:

- The OpenClaw install runbook (per OS)
- The full skill library (~50 deploy-* skills, plus authoring guides)
- Identity/Soul/Heartbeat templates with **Codex GPT-5.x as the default model stack** (not Claude/Anthropic)
- A sanitized reference snapshot of specialized v2 skills from a prior VC-firm engagement — kept as a starting point for custom office deployments, not as the default

## What This Is NOT

- Not the OAC (OpenAgent Connect) backend — that lives in `RShuken/openagent-connect`
- Not a marketing site — that's `openclawinstall`
- Not a place for client profiles or secrets
- Research baseline is the April 2026 audit (`audit/skill-audit-2026-04-19.md`). Re-verify against current OpenClaw before any production deployment.

## Layout

```
openclaw-base/
├── install/                          OS-specific install runbooks
├── skills/                           Deploy-* skill library (flat, mirrors original layout)
│   ├── _authoring/                   How to write new skills (read these first)
│   ├── _archive/                     Older v1s superseded by v2s
│   └── _enterprise-v2-reference/     Sanitized v2 reference snapshot (REFERENCE ONLY, use placeholders)
├── templates/                        Scaffolds for IDENTITY/SOUL/HEARTBEAT/openclaw.json
├── audit/                            Skill audit ledger + research (baseline, model routing, memory options)
└── catalog.json                      Intent-based skill metadata
```

## Status

| Phase | What | State |
|-------|------|-------|
| 1 | Carve from prior agent workspace + sanitize reference engagement | DONE 2026-04-19 |
| 2 | Audit every skill against current OpenClaw, Codex-strip pass | DONE 2026-04-20 |
| 3 | Scrub identifiers, publish to GitHub | DONE 2026-04-20 |
| 4 | Pull on target machine, run install, verify | per-deployment |
| 5 | Fork and customize for a named agent deployment | per-deployment |

## How To Use After Audit

1. Read `install/openclaw-install.md` first
2. Then read `skills/_authoring/_deploy-common.md` for deployment philosophy
3. Pick skills to deploy in order: identity → security-safety → messaging-setup → memory → then everything else
4. Every skill has a verification checklist — done is not done until it passes
