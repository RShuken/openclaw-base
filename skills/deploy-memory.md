# Deploy Memory System

## Compatibility

- **OpenClaw Version**: 2026.4.15+ (compat header refreshed 2026-04-20 — live-verified on 4C)
- **Status**: WORKING — one of the most reliable skills in the library
- **Working commands used by this skill**: `openclaw memory status --json`, `openclaw memory index --force`, `openclaw memory search`, `openclaw health` (replaces the never-existed `openclaw status`), `openclaw doctor --fix`
- **Approach**: uses OpenClaw's built-in `memory` subsystem (sqlite-vec + auth-profiles-backed embeddings). `auth-profiles.json` is where embedding API keys MUST live — NOT `.env` (the built-in memory system doesn't read `.env`).
- **Platform**: macOS and Linux both supported via platform-appropriate service restart (launchctl / systemctl).

## Purpose

Enable OpenClaw's built-in cross-session persistent memory. Without memory configured, each Telegram topic, DM thread, and chat session is fully isolated — the agent cannot recall anything said in a different session. Memory provides vector-searchable context that persists across all sessions.

**When to use:** After OpenClaw is installed and an embedding-capable API key is available (Gemini, OpenAI, Voyage, or Mistral).

**What this skill does:**
1. Discovers current memory status and identifies why it's not working
2. Configures the embedding provider auth so OpenClaw's memory system can generate vectors
3. Indexes workspace memory files
4. Restarts services so live sessions have memory active
5. Verifies vector search works across sessions

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/.openclaw/workspace` |
| `${OPENCLAW_DIR}` | Client profile: config path | `~/.openclaw` |
| `${EMBEDDING_KEY}` | Client's `.env` or provider dashboard | `AIza...` (Gemini) or `sk-...` (OpenAI) |
| `${EMBEDDING_PROVIDER}` | Adaptation point decision | `google`, `openai`, `voyage`, `mistral` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**

Check current memory status (the most informative single command):

**Remote:**
```
openclaw memory status --json 2>&1
```

Key fields to check in the output:
- `provider` — if `"none"`, memory has no embedding provider (the usual problem)
- `requestedProvider` — what OpenClaw tried to use (usually `"auto"`)
- `custom.providerUnavailableReason` — exact error explaining why auto-detection failed
- `vector.available` — whether sqlite-vec extension is loaded
- `files` / `chunks` — how much is indexed (0/0 means nothing indexed)
- `fts.available` — FTS5 status (often `false`; not a blocker if vector works)

Check which embedding API keys exist:

**Remote:**
```
cat ${OPENCLAW_DIR}/agents/main/agent/auth-profiles.json 2>&1
```

**Critical distinction:** OpenClaw's memory system looks for API keys in `auth-profiles.json`, NOT in `~/.openclaw/.env`. The `.env` file is used by custom scripts (KB, CRM, etc.), but the built-in memory system only checks auth profiles. This is the most common reason memory doesn't work.

Check what files exist in the memory source directory:

**Remote:**
```
ls ${WORKSPACE}/memory/
```

## Adaptation Points

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Embedding provider | Google `gemini-embedding-001` (3072 dims) | Client prefers OpenAI `text-embedding-3-small` (1536 dims), Voyage, or Mistral |
| Auth profile key | `google:default` | Different provider — use `openai:default`, `voyage:default`, or `mistral:default` |
| API key source | `GOOGLE_GEMINI_API_KEY` from `~/.openclaw/.env` | Different provider key name or location |
| Memory sources | `memory` (workspace memory dir) | Client wants additional indexed directories |
| FTS5 | Skip (use vector search only) | Client's SQLite has FTS5 compiled in — hybrid search is better but not required |

## Prerequisites

- OpenClaw installed and running (`openclaw health` shows gateway + node running)
- An embedding API key available (Gemini, OpenAI, Voyage, or Mistral)
- sqlite-vec extension available (ships with OpenClaw 2026.2+; check `vector.available` in memory status)

## What Gets Installed

### Auth Profile Entry

A new profile in `${OPENCLAW_DIR}/agents/main/agent/auth-profiles.json` that provides the embedding API key to the memory system.

### Memory Index

SQLite database at `${OPENCLAW_DIR}/memory/main.sqlite` containing:
- File metadata (path, hash, last indexed)
- Content chunks from workspace memory files
- Vector embeddings (dimension depends on provider: Gemini=3072, OpenAI=1536)
- Embedding cache for deduplication

### How Memory Works

- The agent's `${WORKSPACE}/memory/` directory is the primary source. Files the agent writes there (conversation notes, session summaries) are automatically indexed.
- Workspace `.md` files (IDENTITY.md, TOOLS.md, SOUL.md, etc.) are injected as system context in every session — they do NOT go through memory search.
- The Knowledge Base (`skills/knowledge-base/`) is a separate tool the agent queries on demand. It is not part of the memory index. Both are available in all sessions.

## Steps

### Phase 1: Configure Embedding Auth

#### 1.1 Read the Existing API Key `[AUTO]`

The embedding API key usually already exists in `~/.openclaw/.env` (used by KB, CRM, etc.). Read it so we can add it to auth-profiles.

**Remote (Gemini — default):**
```
grep GOOGLE_GEMINI_API_KEY ${OPENCLAW_DIR}/.env
```

**Remote (OpenAI — if adapting):**
```
grep OPENAI_API_KEY ${OPENCLAW_DIR}/.env
```

Expected: A line containing the API key. If not found, the client needs to obtain one first.

If this fails: The key might be stored elsewhere or not yet configured. Check the client profile for API key locations.

#### 1.2 Add Auth Profile `[GUIDED]`

Add the embedding provider to auth-profiles.json. This is the critical step — OpenClaw's memory system only checks this file for API keys.

**Remote (use python3 to safely edit JSON):**
```python
python3 -c "
import json, os

# Read the existing API key
env_path = os.path.expanduser('${OPENCLAW_DIR}/.env')
key = None
with open(env_path) as f:
    for line in f:
        if line.startswith('GOOGLE_GEMINI_API_KEY='):
            key = line.strip().split('=', 1)[1]
            break

if not key:
    print('ERROR: Key not found')
    exit(1)

# Add to auth-profiles.json
auth_path = os.path.expanduser('${OPENCLAW_DIR}/agents/main/agent/auth-profiles.json')
with open(auth_path) as f:
    auth = json.load(f)

auth['profiles']['google:default'] = {
    'type': 'token',
    'provider': 'google',
    'token': key
}

with open(auth_path, 'w') as f:
    json.dump(auth, f, indent=2)

print('Added google:default profile')
print('Profiles:', list(auth['profiles'].keys()))
"
```

> **Adapt for other providers:** Replace `GOOGLE_GEMINI_API_KEY` with the appropriate env var, and `google:default`/`google` with `openai:default`/`openai`, `voyage:default`/`voyage`, or `mistral:default`/`mistral`.

Expected: Auth profiles now include the embedding provider alongside any existing profiles (e.g., `anthropic:default`).

If this fails: Check that `auth-profiles.json` exists and is valid JSON. If corrupted, back it up and create a fresh one with `{"version": 1, "profiles": {}, "usageStats": {}}`.

**Do NOT edit `openclaw.json` to set memory config keys.** OpenClaw has strict schema validation. Keys like `memory.provider`, `memory.sources`, and `memory.extraPaths` are NOT valid config keys and will cause ALL CLI commands to fail with "Unrecognized keys" errors. The memory system auto-detects the provider from auth-profiles.json when `requestedProvider` is `auto` (the default).

### Phase 2: Index and Activate

#### 2.1 Index Memory Files `[AUTO]`

Force a full reindex so the memory system picks up the new embedding provider and indexes all existing files.

**Remote:**
```
openclaw memory index --force 2>&1
```

Expected: "Memory index updated (main)." FTS warnings (`fts unavailable: no such module: fts5`) are normal and harmless — vector search handles retrieval.

If this fails:
- **"Config invalid"**: You edited openclaw.json with invalid keys. Remove them with `openclaw doctor --fix` or manually delete the invalid keys from `memory` section.
- **API key errors**: The auth profile wasn't added correctly. Re-check auth-profiles.json.
- **Quota exhausted**: Gemini free tier has daily embedding limits. Wait for quota reset or switch to a paid tier.

#### 2.2 Restart Gateway and Node `[AUTO]`

Restart both services so live sessions have the memory system active.

**Remote (macOS — launchd):**
```
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/ai.openclaw.gateway.plist 2>&1
sleep 2
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.openclaw.gateway.plist 2>&1
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/ai.openclaw.node.plist 2>&1
sleep 2
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.openclaw.node.plist 2>&1
```

**Remote (Linux — systemd):**
```
systemctl --user restart openclaw-gateway openclaw-node
```

**Remote (Windows — schtasks):**
```
schtasks /End /TN "OpenClaw Gateway" & schtasks /Run /TN "OpenClaw Gateway"
schtasks /End /TN "OpenClaw Node" & schtasks /Run /TN "OpenClaw Node"
```

Expected: Services restart without errors. Gateway and node PIDs change.

If this fails: Check service registration. Use `launchctl list | grep openclaw` (macOS) or `systemctl --user status openclaw-gateway` (Linux) to verify services are registered.

## Verification

**Remote — check memory status shows provider and indexed content:**
```
openclaw memory status --json 2>&1
```

Verify these fields:
- `provider`: should be `"gemini"` (or `"openai"`, etc.)
- `model`: should be `"gemini-embedding-001"` (or provider-specific model)
- `vector.available`: `true`
- `files`: >= 1
- `chunks`: >= 1
- `custom.searchMode`: `"hybrid"` (or `"vector-only"` if FTS unavailable)

**Remote — test vector search returns results:**
```
openclaw memory search --query "test search" 2>&1
```

Expected: Results with similarity scores (e.g., `0.528 memory/2026-03-01.md:1-7`). If no memory files exist yet, create a test note first:

**Remote:**
```
echo "# Test Memory Note\nThis is a test to verify memory indexing works." > ${WORKSPACE}/memory/test.md
openclaw memory index --force 2>&1
openclaw memory search --query "test memory verify" 2>&1
rm ${WORKSPACE}/memory/test.md
openclaw memory index --force 2>&1
```

**Remote — verify full status shows memory active:**
```
openclaw health 2>&1
```

The Memory line should show files, chunks, vector status, and cache count instead of `0 files · 0 chunks · vector unknown`.

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `provider: "none"` after adding auth profile | Auth profile key name doesn't match expected provider format | Verify profile key is exactly `google:default`, `openai:default`, etc. Check `custom.providerUnavailableReason` in status JSON for the exact error. |
| "Unrecognized keys: memory.provider" | Someone edited openclaw.json with invalid memory keys | Run `openclaw doctor --fix` to remove invalid keys. Or manually edit the JSON to clear the `memory` section. |
| `vector.available: false` | sqlite-vec extension not found | Upgrade OpenClaw (`npm update -g openclaw`). sqlite-vec ships with 2026.2+. Check `vector.extensionPath` in status. |
| `files: 0, chunks: 0` after indexing | No files in workspace memory directory | Create at least one `.md` file in `${WORKSPACE}/memory/`. The agent creates these during conversations. |
| "fts unavailable: no such module: fts5" | SQLite wasn't compiled with FTS5 | Not a blocker. Vector search works fine without FTS5. Search mode falls back to `vector-only`. |
| Gemini quota exhausted | Free tier daily embedding limit | Wait for quota reset (resets daily), reduce batch sizes, or upgrade to paid Gemini API. |
| Memory search returns no results | Index is empty or stale | Run `openclaw memory index --force`. Check that `${WORKSPACE}/memory/` has `.md` files. |
| Auth profiles corrupted | Manual JSON editing introduced syntax error | Validate with `python3 -c "import json; json.load(open('path/to/auth-profiles.json'))"`. Fix syntax or restore from backup. |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Read API key | Execute silently | Execute silently | Confirm |
| 1.2 Add auth profile | Execute silently | Confirm before editing | Show JSON changes, confirm |
| 2.1 Index memory | Execute silently | Execute silently | Confirm |
| 2.2 Restart services | Execute silently | Confirm before restart | Confirm each service |

## Dependencies

- **Depends on:** `openclaw/install.md` (OpenClaw must be installed with gateway + node running)
- **Enhanced by:** `deploy-knowledge-base.md` (KB provides queryable external knowledge alongside memory's cross-session context), `deploy-personal-crm.md` (CRM data accessible from any session via tools)
- **Required by:** Any workflow where the agent needs to recall information across Telegram topics, DM threads, or other chat sessions
