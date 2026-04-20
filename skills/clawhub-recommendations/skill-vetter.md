---
name: "skill-vetter"
clawhub_slug: "@spclaudehome/skill-vetter"
clawhub_url: "https://clawhub.ai/@spclaudehome/skill-vetter"
category: "security"
author: "@spclaudehome"
downloads_last_checked: "unverified"
version_last_verified: "unverified"
last_checked: "2026-04-20"
maintenance_status: "unverified"
license: "Unknown <VERIFY>"
recommends_over: "none (net-new capability; no dedicated deploy-* skill)"
cost_tier: "free-install"
privacy_tier: "local"
docs_urls:
  - "https://clawhub.ai/@spclaudehome/skill-vetter"
---

# skill-vetter

## What it is

Static-analysis / security-review skill for other ClawHub skills. Scans a skill's files for known-malicious patterns (crypto-key exfil, SSH key theft, over-scoped permissions, data-exfil URLs). Baseline §L recommends installing **before** any other ClawHub skill.

## Why we recommend it

- Baseline §K #8: **"341–386 malicious skills found on ClawHub Feb 1–3, 2026"** (crypto-exchange API exfil, SSH key theft).
- §L explicitly: "Install Clawdex / skill vetter BEFORE adding ClawHub skills — pre-install + retroactive scanning."
- Cheap to run; catches a real class of attacks cold.

## When to pick this over our own deploy-*

- Always. Make it the first skill installed on any client machine.
- Retroactively scan existing skill inventories on any enrollment you inherit.

## When NOT to pick this

- Only if the client runs exclusively first-party, author-signed skills — still recommended as a defensive layer.

## Install

```bash
clawhub install @spclaudehome/skill-vetter
```

## Configure / wire into OpenClaw

Run pre-install:

```bash
skill-vetter scan <slug-or-path>
```

Optional: wire a hook that runs `skill-vetter` on any `openclaw skills install` invocation. See `../_authoring/_deploy-common.md` for hook patterns; or enable via the skill's own hook if it ships one.

Retroactive scan of existing skills:

```bash
skill-vetter scan --all ~/.openclaw/skills/
```

## Known issues / gotchas

- Static analysis is necessarily incomplete — treat as a filter, not a guarantee.
- Verify the vetter itself with `skill-vetter scan @spclaudehome/skill-vetter` (yes, recursively). Baseline doesn't verify @spclaudehome's trust status — do your own review.
- Author handle `@spclaudehome` is as given in the user's task spec; verify on clawhub.ai.

## Citations

- https://clawhub.ai/@spclaudehome/skill-vetter
- `audit/_baseline.md` §K, §L, §O
