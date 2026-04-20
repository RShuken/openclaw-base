# Token Optimization

## Purpose

Configure an OpenClaw client to minimize token costs without sacrificing quality. Sets up a cheap default model, powerful on-demand escalation, model aliases, fallback chains, prompt caching awareness, and optional local heartbeat.

**When to use:** During initial client setup, or when a client reports high token costs / wants to optimize spending.

## Prerequisites

- OpenClaw agent installed and running on the client machine
- At least one provider configured (OpenRouter, OpenAI Codex, Anthropic, etc.)
- Access to the client's config file:
  - **Windows:** `C:\Users\<user>\.openclaw\openclaw.json`
  - **macOS/Linux:** `~/.openclaw/openclaw.json`

## Steps

### 1. Detect Current Provider Setup

Check which providers are already configured:

```
openclaw config show
```

Or read the config file directly:

- **Windows (PowerShell):** `type $env:USERPROFILE\.openclaw\openclaw.json`
- **macOS/Linux:** `cat ~/.openclaw/openclaw.json`

Look for the `providers` section. Common setups:

| Provider | Type | Notes |
|----------|------|-------|
| `openrouter` | API key | Pay-per-token, routes to many models |
| `openai-codex` | OAuth | Codex Plus subscription, includes GPT-5.x models |
| `anthropic` | API key | Claude models, explicit cache control |
| `local` (Ollama) | Local | Free, for heartbeat/keep-alive only |

### 2. Choose a Cheap Default Model

The default model handles all routine tasks (status checks, simple commands, formatting). Pick the cheapest option available from the client's providers:

| Provider | Recommended Cheap Default | Cost |
|----------|--------------------------|------|
| OpenRouter | `openrouter/openai/gpt-4.1-nano` | ~$0.005/session |
| OpenRouter | `openrouter/anthropic/claude-haiku-4-5` | ~$0.01/session |
| Anthropic | `anthropic/claude-haiku-4-5` | ~$0.01/session |

Set the default in the config:

```json
{
  "models": {
    "default": "<chosen-cheap-model>"
  }
}
```

### 3. Set the Primary (Powerful) Model

This is the model used when the operator explicitly escalates for hard problems:

| Provider | Recommended Primary | Notes |
|----------|-------------------|-------|
| OpenAI Codex | `openai-codex/gpt-5.3-codex` | Subscription-based, no per-token cost |
| OpenRouter | `openrouter/openai/gpt-4o` | Pay-per-token, strong mid-tier |
| Anthropic | `anthropic/claude-sonnet-4-6` | Strong reasoning, explicit caching |

```json
{
  "models": {
    "default": "<cheap-model>",
    "primary": "<powerful-model>"
  }
}
```

### 4. Configure the Fallback Chain

Fallbacks ensure continuity when the primary model is rate-limited or down. Order from most capable to cheapest:

```json
{
  "models": {
    "default": "<cheap-model>",
    "primary": "<powerful-model>",
    "fallbacks": [
      "<next-best-model>",
      "<mid-tier-model>",
      "<auto-routed-option>",
      "<cheap-model>"
    ]
  }
}
```

**Example (OpenRouter + Codex Plus):**

```json
{
  "fallbacks": [
    "openai-codex/gpt-5.2",
    "openai-codex/gpt-5.1-codex-max",
    "openrouter/auto",
    "openrouter/openai/gpt-4o",
    "openrouter/openai/gpt-4.1-nano"
  ]
}
```

**Example (Anthropic):**

```json
{
  "fallbacks": [
    "anthropic/claude-sonnet-4-6",
    "anthropic/claude-haiku-4-5"
  ]
}
```

### 5. Set Up Model Aliases

Aliases let the operator switch models with short names:

```json
{
  "aliases": {
    "nano": "<cheap-model-id>",
    "codex": "<powerful-model-id>",
    "mid": "<mid-tier-model-id>",
    "auto": "<auto-routed-model-id>"
  }
}
```

Usage during a session:
```
/model nano       # Switch to cheap (routine tasks)
/model codex      # Switch to powerful (hard problems)
/model mid        # Switch to mid-tier
/model auto       # Let the router pick
```

**Rule:** Start on the cheap model. Escalate only when quality is insufficient. Switch back when done.

### 6. Prompt Caching Awareness

Caching reduces costs on repeated prompt prefixes (system prompts, session context).

| Provider | Caching | Config Needed | Discount |
|----------|---------|---------------|----------|
| OpenAI / OpenRouter | Automatic | None | 50% on cached input tokens |
| Anthropic | Explicit | `cache_control` headers | 90% on cached input tokens |

**For OpenAI/OpenRouter:** No action needed. Caching activates automatically after the first request in a session. Cache persists ~5-10 minutes between requests.

**For Anthropic:** Add cache breakpoints to the system prompt. See Anthropic's documentation for `cache_control` header placement. Minimum cacheable prefix: 1024 tokens (Sonnet), 2048 tokens (Haiku).

### 7. Heartbeat / Keep-Alive Setup (Optional)

For zero-cost heartbeat pings, use a local model via Ollama instead of burning cloud tokens.

#### Installing Ollama

**Windows (PowerShell as Admin):**
```powershell
winget install Ollama.Ollama
```

**macOS (Homebrew):**
```bash
brew install ollama
```

**Linux:**
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

#### Pull a Lightweight Model

```
ollama pull tinyllama
```

`tinyllama` is ~637MB, runs on CPU. Alternative: `ollama pull phi3:mini`

#### Configure Heartbeat

```json
{
  "providers": {
    "local": {
      "type": "ollama",
      "baseUrl": "http://localhost:11434"
    }
  },
  "heartbeat": {
    "model": "local/tinyllama",
    "intervalSeconds": 30,
    "prompt": "ping",
    "maxTokens": 1
  }
}
```

**Savings:** 2,880 requests/day at 30s intervals. Remote (nano) = ~$0.03/day. Local (Ollama) = $0.00/day.

## Verification

Run these commands after applying the config changes:

```
# Check current config
openclaw config show

# Verify default model
openclaw model current
# Expected: your chosen cheap default

# Verify aliases are set
openclaw model list
# Should show all configured aliases

# Verify fallback chain
openclaw model fallbacks
# Should list all fallback models in order

# Test provider connectivity
openclaw provider test <provider-name>
# Expected: OK - <model> responded in Xms

# Test heartbeat (if configured)
ollama list                  # Confirm Ollama is running
openclaw heartbeat test      # Expected: OK - local/tinyllama responded in Xms

# Verify prompt caching (OpenAI/OpenRouter only)
# After 2+ requests with same prompt prefix:
openclaw debug last-response
# Look for: cached_tokens > 0 in usage breakdown

# Verify Telegram (if configured)
openclaw channel test telegram
```

## Platform Notes

### Windows (PowerShell 5.1)
- No `&&` operator — use `;` to chain commands
- `$_` is consumed by PowerShell — use backtick escape (`` `$_ ``) or alternate syntax
- Ollama installs as a Windows service, runs automatically
- Config path: `C:\Users\<user>\.openclaw\openclaw.json`
- Check Ollama via system tray icon

### macOS
- Config path: `~/.openclaw/openclaw.json`
- Ollama: `brew services start ollama` to run as background service
- `launchctl` can be used for agent auto-start

### Linux
- Config path: `~/.openclaw/openclaw.json`
- Ollama: `systemctl enable ollama && systemctl start ollama`
- Use `systemd` for agent auto-start

## Cost Reference

| Model | Input (per 1M tokens) | Output (per 1M tokens) | ~Cost per 20-exchange session |
|-------|-----------------------|------------------------|-------------------------------|
| `gpt-4.1-nano` | $0.10 | $0.40 | $0.005 |
| `gpt-4o` | $2.50 | $10.00 | $0.12 |
| `claude-haiku-4-5` | $0.80 | $4.00 | $0.04 |
| `claude-sonnet-4-6` | $3.00 | $15.00 | $0.15 |
| `gpt-5.3-codex` | (subscription) | (subscription) | $0 marginal |
