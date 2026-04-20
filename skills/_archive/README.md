# OpenAgent Connect — Skills

Reusable runbooks for client setup and operations. Each skill is a self-contained
markdown procedure any agent can follow.

## Available Skills

| Skill | Description |
|-------|-------------|
| [setup-windows.md](setup-windows.md) | Set up OpenClaw agent on Windows 10/11 |
| [setup-macos.md](setup-macos.md) | Set up OpenClaw agent on macOS (Intel + Apple Silicon) |
| [setup-ubuntu-gce.md](setup-ubuntu-gce.md) | Set up OpenClaw agent on Ubuntu GCE VM |
| [token-optimization.md](token-optimization.md) | Configure model tiers, aliases, and cost optimization |

## Skill Format

Each skill follows this structure:
- **Purpose** — What this skill does and when to use it
- **Prerequisites** — What needs to be in place first
- **Steps** — Numbered procedure with exact commands
- **Verification** — How to confirm it worked
- **Platform Notes** — Windows/macOS differences where applicable
