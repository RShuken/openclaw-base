---
name: "gog (Google Workspace CLI)"
clawhub_slug: "@steipete/gog"
clawhub_url: "https://clawhub.ai/@steipete/gog"
category: "integrations"
author: "@steipete"
downloads_last_checked: "158k (per audit/_baseline.md §B)"
version_last_verified: "unverified"
last_checked: "2026-04-20"
maintenance_status: "active"
license: "Unknown <VERIFY>"
recommends_over: "deploy-google-workspace.md"
cost_tier: "free-install"
privacy_tier: "cloud"
docs_urls:
  - "https://clawhub.ai/@steipete/gog"
---

# gog — Google Workspace CLI by @steipete

## What it is

`gog` is the community-standard CLI for wiring an OpenClaw agent into Google Workspace — Gmail, Calendar, Drive, Docs, Sheets. It ships with a managed OAuth flow, keyring-backed token storage, and idiomatic subcommands that the agent can call directly.

## Why we recommend it

- Most-installed Google Workspace integration on ClawHub (158k downloads per baseline).
- Managed OAuth with `gog auth keyring file` — side-steps macOS Keychain prompts on headless enrollments.
- Maintained by @steipete, who ships quickly and responds to issues.
- Our own `deploy-google-workspace.md` essentially wraps this CLI anyway.

## When to pick this over our own deploy-*

- Always prefer `@steipete/gog` install as the primary path. Our deploy-google-workspace.md is now more of a configuration recipe around the gog CLI than a from-scratch build.
- Any greenfield client install should just consume gog directly.

## When NOT to pick this

- Client needs a hard-audited, self-hosted Google integration with zero third-party CLI dependency. Rare; would require building from `deploy-google-workspace.md` scratch.
- Client runs an air-gapped or non-macOS environment where gog's keyring assumptions break. Verify on target.

## Install

```bash
clawhub install @steipete/gog
# Verify exact CLI form on target OpenClaw version (2026.4.15+ uses `openclaw skills install` in some surfaces).
```

## Configure / wire into OpenClaw

After install:

```bash
gog auth keyring file   # avoids macOS Keychain GUI prompts
gog auth login          # OAuth flow
```

Known issue: remote OAuth state-mismatch during headless install — see the `expect` workaround in `../deploy-google-workspace.md`.

## Known issues / gotchas

- Remote-OAuth state mismatch bug on some enrollments (workaround documented in deploy-google-workspace.md).
- Keyring defaults differ per OS — confirm `gog auth keyring list` before running OAuth.

## Citations

- https://clawhub.ai/@steipete/gog
- `skills/deploy-google-workspace.md`
- `audit/_baseline.md` §B
