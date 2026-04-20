# OpenClaw Setup — macOS

## Purpose

Set up and connect an OpenClaw agent on a macOS machine (Ventura 13, Sonoma 14, Sequoia 15 — Intel and Apple Silicon). Covers the full flow from running the one-liner through troubleshooting every common failure mode.

**When to use:** First-time client onboarding on macOS, or when reconnecting a macOS client whose agent isn't running.

## Prerequisites

- Operator has generated a one-liner via the `assist` command
- Client machine has internet access to `https://api.openclawinstall.net`
- Client can open Terminal (built into every Mac)

**No software installation is required on the client.** The one-liner downloads a self-contained bundle with a portable Node.js binary (architecture-matched), the agent, and native binaries. No Homebrew, no nvm, nothing.

## Steps

### 1. Have the Client Open Terminal

Tell them: "Press Cmd+Space, type `Terminal`, press Enter."

### 2. Paste the One-Liner

The one-liner looks like:

```bash
curl -fsSL "https://api.openclawinstall.net/i/m/<TOKEN>/<CLIENT_ID>" | bash
```

Tell the client to paste it into Terminal and press Enter.

**What happens behind the scenes:**
1. `curl` fetches the installer script from the server
2. The script detects the Mac's architecture (`uname -m` → `arm64` or `x86_64`)
3. Downloads the matching bundle ZIP (~33MB): `macos-arm64.zip` or `macos-x64.zip`
4. Extracts to `/tmp/openagent-agent`
5. Runs `chmod +x` on the bundled `node` binary
6. Executes `./node agent.js --server ... --token ... --client ...`
7. Agent connects to the control plane via WebSocket (WSS on port 443)

### 3. Read Back the Verification Code

The client's Terminal will show:

```
Session: abc123-def456
Read this code to operator: 847291
Connector online. Press Ctrl+C to stop.
```

Have the client read you the **session ID** and **6-digit code**. Enter them in the operator prompt.

### 4. Verify the Connection

Once verified, test with a simple command:

```
openagent> whoami
openagent> hostname
openagent> sw_vers
openagent> uname -m
```

## Troubleshooting

### Gatekeeper: "node is damaged and can't be opened"

**Symptom:** Error dialog: `"node" is damaged and can't be opened` or `"node" can't be opened because Apple cannot check it for malicious software`.

**Why:** The bundled `node` binary has the `com.apple.quarantine` extended attribute set. This normally does NOT happen when downloaded via `curl` in Terminal (only browser downloads set quarantine), but it can happen on Sequoia (macOS 15) with tightened Gatekeeper rules.

**Fix:**
```bash
# Remove quarantine flag from the entire agent directory
xattr -dr com.apple.quarantine /tmp/openagent-agent/

# Then re-run the agent
cd /tmp/openagent-agent && ./node agent.js --server ... --token ... --client ...
```

If that still fails (Sequoia): Go to **System Settings > Privacy & Security**, scroll down, and click **"Allow Anyway"** next to the blocked binary. Then re-run.

### "Allow incoming network connections" Dialog

**Symptom:** macOS shows: "Do you want the application 'node' to accept incoming network connections?"

**Does this affect the agent?** No. The agent only makes **outbound** WebSocket connections. This dialog is about **inbound** connections (listening servers). The agent works regardless of whether you click Allow or Deny.

**Recommendation:** Click **Allow** anyway — it won't hurt, and prevents confusion if terminal commands launched through the agent start a server.

**If the client accidentally clicked Deny and it causes issues:**
```bash
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --remove /tmp/openagent-agent/node
```

### Architecture Mismatch

**Symptom:** Agent crashes with: `(mach-o file, but is an incompatible architecture (have 'x86_64', need 'arm64'))` or vice versa.

**Why:** The installer script auto-detects architecture via `uname -m`. This can fail if:
- Terminal.app was opened with "Open using Rosetta" checked (makes `uname -m` return `x86_64` on an Apple Silicon Mac)
- The client is using an x86_64 terminal emulator

**Diagnose:**
```bash
# Check actual hardware
sysctl -n machdep.cpu.brand_string   # Shows actual CPU
uname -m                               # Shows what the shell sees
sysctl -n sysctl.proc_translated 2>/dev/null
# 0 = native, 1 = running under Rosetta
```

**Fix:**
```bash
# Force native arm64 shell
arch -arm64 zsh

# Re-run the one-liner in this shell
curl -fsSL "https://api.openclawinstall.net/i/m/<TOKEN>/<CLIENT_ID>" | bash
```

### macOS Firewall Blocking Outbound Connections

**Symptom:** Agent starts but can't connect. "Connector error: connect ETIMEDOUT".

**Built-in macOS firewall:** OFF by default, and even when enabled, it only blocks **inbound** connections. Outbound WSS is never blocked by the built-in firewall.

**Third-party firewalls (Little Snitch, Lulu, Vallum):** These DO filter outbound connections.

**Fix for Little Snitch / Lulu:** When the connection dialog appears, click **Allow** for the `node` binary connecting to `api.openclawinstall.net` on port 443.

**Check firewall status:**
```bash
/usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
# "Firewall is disabled. (State = 0)" = fine
```

### /tmp Cleanup Deletes the Agent

**Symptom:** Agent was working yesterday, today it's gone. "Cannot find module '/tmp/openagent-agent/agent.js'".

**Why:** macOS periodically cleans `/private/tmp` (every 3 days by default). `/tmp` is a symlink to `/private/tmp`.

**Fix:** Re-run the one-liner. Or install to a persistent path:
```bash
AGENT_DIR="$HOME/.openagent/agent"
mkdir -p "$AGENT_DIR"
# Extract the bundle there instead of /tmp
```

### Connection Drops on Sleep/Wake

**Symptom:** Agent was connected, Mac went to sleep (lid close, idle timeout), agent shows "Connector disconnected" after waking.

**Why:** macOS suspends all processes during sleep. Network interfaces go down, TCP connections drop, the WebSocket closes. The agent currently does not auto-reconnect.

**Prevention:**
```bash
# Run the agent with caffeinate to prevent idle sleep
caffeinate -i ./node agent.js --server ... --token ... --client ...
# -i = prevent idle sleep (display can still sleep)
# -s = prevent system sleep entirely (even on AC)
```

**Power settings (keep Mac awake on AC power):**
```bash
sudo pmset -c sleep 0           # Never sleep on charger
sudo pmset -c displaysleep 0    # Never turn off display on charger
```

**After sleep/wake:** Re-run the one-liner to establish a new session.

## Persistent Agent (Optional)

### launchd LaunchAgent (recommended)

Create `~/Library/LaunchAgents/com.openagent.agent.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.openagent.agent</string>

    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/caffeinate</string>
        <string>-i</string>
        <string>/tmp/openagent-agent/node</string>
        <string>/tmp/openagent-agent/agent.js</string>
        <string>--server</string>
        <string>https://api.openclawinstall.net</string>
        <string>--token</string>
        <string>REPLACE_TOKEN</string>
        <string>--client</string>
        <string>REPLACE_CLIENT_ID</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <dict>
        <key>NetworkState</key>
        <true/>
    </dict>

    <key>StandardOutPath</key>
    <string>/tmp/openagent-agent.stdout.log</string>

    <key>StandardErrorPath</key>
    <string>/tmp/openagent-agent.stderr.log</string>

    <key>ThrottleInterval</key>
    <integer>10</integer>
</dict>
</plist>
```

**Important:** `ProgramArguments` must use **absolute paths**. `~/` does NOT expand. nvm paths do NOT work.

**Load the service:**
```bash
# macOS 13+ (modern)
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.openagent.agent.plist

# Stop
launchctl bootout gui/$(id -u)/com.openagent.agent

# Legacy (still works)
launchctl load ~/Library/LaunchAgents/com.openagent.agent.plist
launchctl unload ~/Library/LaunchAgents/com.openagent.agent.plist

# Check status
launchctl list | grep openagent
```

### tmux (simplest, doesn't survive reboot)

```bash
tmux new-session -d -s openagent \
  'cd /tmp/openagent-agent && ./node agent.js --server https://api.openclawinstall.net --token TOKEN --client CLIENT_ID'

# Reattach later
tmux attach -t openagent
```

## Verification

After the agent is connected, run these to confirm everything is healthy:

```
openagent> whoami
openagent> hostname
openagent> sw_vers
openagent> uname -m
openagent> sysctl -n sysctl.proc_translated 2>/dev/null || echo "native"
openagent> curl -sS -o /dev/null -w "%{http_code}" https://api.openclawinstall.net/health
```

## Quick Diagnostic Script

Run this on the client Mac before starting a session to catch common issues:

```bash
echo "=== macOS Version ===" && sw_vers
echo "=== Architecture ===" && uname -m
echo "=== Rosetta ===" && (sysctl -n sysctl.proc_translated 2>/dev/null && echo "Running under Rosetta" || echo "Native")
echo "=== Firewall ===" && /usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
echo "=== Shell ===" && echo "$SHELL"
echo "=== Power Settings ===" && pmset -g | grep -E "sleep|displaysleep"
echo "=== Disk Space ===" && df -h / | tail -1
echo "=== Connectivity ===" && curl -sS -o /dev/null -w "Health check: HTTP %{http_code}\n" https://api.openclawinstall.net/health
```

## Quick Reference

```
One-liner:      curl -fsSL "https://api.openclawinstall.net/i/m/<TOKEN>/<CLIENT_ID>" | bash
Bundle path:    /tmp/openagent-agent/
Shell:          zsh (default since Catalina)
Architecture:   Detected automatically (arm64 for Apple Silicon, x64 for Intel)
Stop agent:     Ctrl+C in the Terminal window
Prevent sleep:  caffeinate -i <command>
```
