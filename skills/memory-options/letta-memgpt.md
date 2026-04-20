---
name: "Letta (formerly MemGPT)"
category: "structured-memory"
repo: "https://github.com/letta-ai/letta"
license: "Apache-2.0"
version_last_verified: "current main"
last_checked: "2026-04-20"
maintenance_status: "active"
openclaw_integration: "none-native (MCP bridge possible)"
cost_tier: "extra-llm-calls"
privacy_tier: "fully-local or hybrid"
requires:
  - "Python 3.11+"
  - "Own LLM + vector store"
docs_urls:
  - "https://github.com/letta-ai/letta"
  - "https://github.com/letta-ai/letta-code"
  - "https://www.letta.com/blog/memgpt-and-letta"
---

# Letta (formerly MemGPT)

## Purpose

Self-editing memory agent framework. Originated the "agent rewrites its own core memory blocks" pattern.

## When to pick this

- You want self-editing memory as a first-class primitive
- Willing to run an alternative agent framework alongside or instead of OpenClaw
- Interested in `letta-code` (memory-first coding harness portable across Claude/GPT/Gemini/GLM/Kimi)

## OpenClaw integration status

**No native OpenClaw plugin as of April 2026.** Integration paths:

1. Call Letta as an MCP server from OpenClaw — but you're running two agent frameworks side-by-side
2. Use Letta as a replacement harness entirely, not as an extension

## Why this is more of a "consider later" than "install now"

- Replacing the agent harness means retraining yourself on Letta's model (core-memory-block abstraction)
- MCP bridge means two agent frameworks, double the surface area
- For most OpenClaw deployments, `mem0.md` gives you 80% of the structured-memory benefit at 10% the operational complexity

## When to pick Letta over staying with OpenClaw

- You're building a new agent from scratch and haven't committed to OpenClaw yet
- You specifically want the self-editing memory pattern as the agent's primary memory mechanism
- You're doing heavy coding work and `letta-code` fits

## Citations

- https://github.com/letta-ai/letta
- https://github.com/letta-ai/letta-code
- https://www.letta.com/blog/memgpt-and-letta
