# OpenClaw Setup — Windows

## Purpose

Set up and connect an OpenClaw agent on a Windows 10/11 machine. Covers the full flow from running the one-liner through troubleshooting every common failure mode.

**When to use:** First-time client onboarding on Windows, or when reconnecting a Windows client whose agent isn't running.

## Prerequisites

- Operator has generated a one-liner via the `assist` command (the one-liner includes the token and client ID)
- Client machine has internet access to `https://api.openclawinstall.net`
- Client can open PowerShell (built into every Windows 10/11 machine)

**No software installation is required on the client.** The one-liner downloads a self-contained bundle (~33MB) with portable Node.js, the agent, and all native binaries. No system Node.js, no npm, nothing.

## Steps

### 1. Have the Client Open PowerShell

Tell them: "Click Start, type `PowerShell`, click **Windows PowerShell**."

**Do NOT use "Run as Administrator"** unless specifically needed for power management steps later. Running as a regular user is safer and sufficient for the agent.

### 2. Paste the One-Liner

The one-liner looks like:

```powershell
irm 'https://api.openclawinstall.net/i/w/<TOKEN>/<CLIENT_ID>' | iex
```

Tell the client to paste it into PowerShell and press Enter.

**What happens behind the scenes:**
1. `irm` (Invoke-RestMethod) fetches the installer script from the server
2. The script downloads a ZIP bundle (~33MB) to `%TEMP%\openagent-agent`
3. The ZIP contains `node.exe` + `agent.js` + `node-pty` native bindings
4. The script extracts the ZIP and runs `node.exe agent.js --server ... --token ... --client ...`
5. The agent connects to the control plane via WebSocket (WSS on port 443)

### 3. Read Back the Verification Code

The client's PowerShell window will show:

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
openagent> systeminfo | findstr /C:"OS Name" /C:"OS Version"
```

## Troubleshooting

### Execution Policy Errors

**Symptom:** Error about "running scripts is disabled on this system."

**Why it happens:** Fresh Windows installs have execution policy set to `Restricted`. However, this almost never triggers with the `irm | iex` one-liner because:
- The `-ExecutionPolicy Bypass` flag is included in the generated script
- `irm | iex` runs inline commands, not a `.ps1` file — execution policy only applies to script files

**When it DOES trigger:** If the client's organization has set execution policy via **Group Policy (GPO)**, the bypass flag is ignored.

**Fix (try in order):**
```powershell
# Option 1: Set policy for current user only (no admin needed)
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser -Force

# Option 2: Set policy for current session only (no admin needed)
Set-ExecutionPolicy Bypass -Scope Process -Force

# Option 3: Run the one-liner through cmd.exe to bypass PowerShell policy entirely
cmd /c "powershell -ExecutionPolicy Bypass -Command \"irm 'https://api.openclawinstall.net/i/w/<TOKEN>/<CLIENT_ID>' | iex\""
```

**If GPO blocks everything:** The client's IT department must whitelist the operation.

### TLS/SSL Errors

**Symptom:** "Could not create SSL/TLS secure channel"

**Why:** PowerShell 5.1 defaults to TLS 1.0, but the server requires TLS 1.2+. The installer script already includes the TLS fix, but if running manually:

```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
```

### Windows Firewall Popup

**Symptom:** A dialog appears: "Windows Defender Firewall has blocked some features of Node.js JavaScript Runtime"

**What to do:** Click **"Allow access"**. This allows `node.exe` (from the downloaded bundle) to make network connections.

**If the client clicked "Cancel":** The agent cannot connect. Fix:
```powershell
# As admin — remove the block rule
netsh advfirewall firewall delete rule name="Node.js JavaScript Runtime"
```

Then re-run the one-liner.

### Antivirus Quarantine

**Symptom:** Agent fails with "file not found" errors for `node.exe` or `pty.node`. Or the one-liner downloads but nothing happens.

**Why:** Windows Defender or third-party antivirus (Norton, McAfee, AVG, Bitdefender) may:
- Quarantine `node.exe` (reputation-based: new/unknown binary)
- Flag `winpty-agent.exe` from the `node-pty` package (heuristic: unknown binary spawning shells)
- Block the `irm | iex` pattern itself (known attack vector)

**Fix for Windows Defender (as admin):**
```powershell
Add-MpExclusion -Path "$env:TEMP\openagent-agent"
Add-MpExclusion -Process "node.exe"
```

**Fix for third-party AV:** Add `%TEMP%\openagent-agent\` to the exclusion list in the antivirus settings UI.

**If `irm | iex` itself is blocked:** Download the script to a file first:
```powershell
$script = Invoke-RestMethod -Uri 'https://api.openclawinstall.net/i/w/<TOKEN>/<CLIENT_ID>'
# Client can inspect $script here
Invoke-Expression $script
```

### SmartScreen Warning

**Symptom:** "Windows protected your PC — Microsoft Defender SmartScreen prevented an unrecognized app from starting"

**Why:** Only happens if the client downloaded the ZIP manually via a browser (Edge/Chrome) instead of using the one-liner. Browser downloads set a `Zone.Identifier` flag.

**Fix:**
```powershell
Unblock-File -Path "$env:TEMP\openagent-agent\node.exe"
# Or: Right-click the ZIP > Properties > check "Unblock" > Apply
```

### Connection Timeout / Hang

**Symptom:** Agent starts but never shows the verification code. Or it shows "Connector error: connect ETIMEDOUT".

**Possible causes:**
1. **Corporate proxy:** Many corporate networks route all traffic through an HTTP proxy that strips WebSocket `Upgrade` headers. WSS on port 443 works better than WS on 80 since TLS traffic can't be inspected.
2. **SSL inspection / MITM proxy:** Corporate proxy terminates TLS and re-signs it. Fix:
   ```powershell
   $env:NODE_TLS_REJECT_UNAUTHORIZED = "0"  # Insecure, last resort
   # Or better: set the corporate root CA
   $env:NODE_EXTRA_CA_CERTS = "C:\path\to\corporate-ca.pem"
   ```
3. **No internet:** Have the client test: `irm https://api.openclawinstall.net/health`

### PowerShell 5.1 Gotchas (During the Session)

Once connected, commands sent by the operator run in the client's PowerShell 5.1. Watch for:

| Issue | Symptom | Fix |
|-------|---------|-----|
| No `&&` operator | `The token '&&' is not a valid statement separator` | Use `;` instead of `&&` |
| `$_` consumed by PS | Variable interpolation eats `$_` in commands | Escape as `` `$_ `` or use single quotes |
| UTF-16LE encoding | Non-ASCII characters appear garbled | Run `[Console]::OutputEncoding = [System.Text.Encoding]::UTF8` first |

## Power Management (Optional)

If the client's machine needs to stay awake for persistent sessions (laptop):

```powershell
# Lid close = Do nothing (AC and battery) — requires admin
powercfg -setacvalueindex SCHEME_CURRENT 4f971e89-eebd-4455-a8de-9e59040e7347 5ca83367-6e45-459f-a27b-476b1d01c936 0
powercfg -setdcvalueindex SCHEME_CURRENT 4f971e89-eebd-4455-a8de-9e59040e7347 5ca83367-6e45-459f-a27b-476b1d01c936 0
powercfg -setactive SCHEME_CURRENT

# Never sleep — requires admin
powercfg /change standby-timeout-ac 0
powercfg /change standby-timeout-dc 0

# Optional: monitor timeout (10min AC, 5min battery)
powercfg /change monitor-timeout-ac 10
powercfg /change monitor-timeout-dc 5
```

## Persistent Agent (Optional)

The one-liner runs the agent in the foreground. For a persistent background agent:

### Task Scheduler (no admin needed)

```powershell
$action = New-ScheduledTaskAction `
  -Execute "$env:TEMP\openagent-agent\node.exe" `
  -Argument "`"$env:TEMP\openagent-agent\agent.js`" --server https://api.openclawinstall.net --token TOKEN --client CLIENT_ID" `
  -WorkingDirectory "$env:TEMP\openagent-agent"
$trigger = New-ScheduledTaskTrigger -AtLogOn
Register-ScheduledTask -TaskName "OpenAgent" -Action $action -Trigger $trigger
```

**Caveat:** Task Scheduler does not auto-restart on crash. If you need crash recovery, use NSSM (requires admin):

### NSSM (admin, auto-restart)

Download `nssm.exe` from https://nssm.cc, then:

```cmd
nssm install OpenAgent "%TEMP%\openagent-agent\node.exe" "%TEMP%\openagent-agent\agent.js" --server https://api.openclawinstall.net --token TOKEN --client CLIENT_ID
nssm set OpenAgent AppDirectory "%TEMP%\openagent-agent"
nssm set OpenAgent AppRestartDelay 5000
nssm start OpenAgent
```

## Verification

After the agent is connected, run these to confirm everything is healthy:

```
openagent> whoami
openagent> hostname
openagent> [System.Environment]::OSVersion.VersionString
openagent> node --version    # Should fail (no system Node needed)
openagent> powershell -Command "$PSVersionTable.PSVersion"
openagent> Test-NetConnection api.openclawinstall.net -Port 443
```

## Quick Reference

```
One-liner:      irm 'https://api.openclawinstall.net/i/w/<TOKEN>/<CLIENT_ID>' | iex
Bundle path:    %TEMP%\openagent-agent\
Shell:          PowerShell 5.1 (powershell.exe)
Chain commands: Use ; (not &&)
Escape $_:      Use `$_ (backtick)
Stop agent:     Ctrl+C in the PowerShell window
```
