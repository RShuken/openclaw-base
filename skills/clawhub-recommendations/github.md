---
name: "github (steipete)"
clawhub_slug: "@steipete/github"
clawhub_url: "https://clawhub.ai/@steipete/github"
category: "integrations"
author: "@steipete"
downloads_last_checked: "unverified"
version_last_verified: "unverified"
last_checked: "2026-04-20"
maintenance_status: "active"
license: "Unknown <VERIFY>"
recommends_over: "none (deleted github-cicd skill was engagement-specific)"
cost_tier: "free-install"
privacy_tier: "cloud"
docs_urls:
  - "https://clawhub.ai/@steipete/github"
---

# github by @steipete

## What it is

Community-standard GitHub integration skill — code review assistance, PR triage, issue triage, basic repo ops. Ships with `gh`-backed auth flow and standardized tool surface for agents.

## Why we recommend it

- @steipete's skills set the bar for structural quality on ClawHub (see §H of baseline for the pattern).
- More narrowly-scoped and battle-tested than our prior `github-cicd` skill (which was engagement-specific).
- Pairs cleanly with `gog` and `notion` from the same author, consistent ergonomics.

## When to pick this over our own deploy-*

- Any new client needing agent-driven GitHub work. We currently have no general-purpose deploy-github-*.md skill.
- For CI/CD-specific automations consider `github-code-review-cicd` or `openclaw-github-assistant` (baseline §B) as alternatives.

## When NOT to pick this

- Client's workflow is fully ticket-driven through Linear/Asana/Shortcut — only reach for GitHub skill if direct repo ops are needed.
- Enterprise GitHub with SSO/SCIM constraints may need a separate org-scoped PAT strategy.

## Install

```bash
clawhub install @steipete/github
# Verify on target; 2026.4.15+ surfaces may use `openclaw skills install`.
```

## Configure / wire into OpenClaw

Requires `gh auth login` or a GitHub PAT in `~/.openclaw/.env`. Follow the skill README — @steipete skills typically document the env-var name explicitly.

## Known issues / gotchas

- PAT scopes matter — `repo`, `read:org`, `workflow` are usually the minimum. Over-scoping is a common security smell flagged by `skill-vetter`.
- GitHub API secondary rate limits can bite heavy automated PR-triage loops.

## Citations

- https://clawhub.ai/@steipete/github
- `audit/_baseline.md` §B
