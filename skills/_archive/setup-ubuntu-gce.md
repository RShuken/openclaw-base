# OpenClaw Setup — Ubuntu on Google Cloud (GCE)

## Purpose

Set up and run an OpenClaw agent on a Google Cloud Compute Engine VM running Ubuntu 22.04 LTS or 24.04 LTS. Unlike Windows/macOS setups (where the client runs a one-liner in a GUI terminal), GCE VMs are headless — this skill covers SSH-safe setup, systemd persistence, memory constraints, and production hardening.

**When to use:** Deploying an always-on OpenClaw agent on a cloud VM, or setting up a client's GCE instance.

## Prerequisites

- A GCE VM running Ubuntu 22.04 LTS or 24.04 LTS
- SSH access to the VM
- The VM has outbound internet access (external IP or Cloud NAT configured)
- Operator has generated a one-liner or has the server URL + token + client ID

## Steps

### 1. SSH into the VM

```bash
gcloud compute ssh <INSTANCE_NAME> --zone <ZONE>
```

### 2. Install System Dependencies

The self-contained bundle approach works on Linux too, but there is no pre-built Linux bundle in the current codebase (only Windows and macOS). For Ubuntu, install Node.js and build from source:

```bash
# Update package lists
sudo apt update

# Install Node.js 20 LTS via NodeSource (NOT the Ubuntu default)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Verify
node --version   # Should show v20.x.x
npm --version    # Should show 10.x.x

# Install build tools for native modules (node-pty requires compilation)
sudo apt install -y build-essential python3 make gcc g++
```

**Why not `apt install nodejs`?**
- Ubuntu 22.04's default repo has Node.js **v12** — far too old
- Ubuntu 24.04's default repo has Node.js **v18** — works but reaches EOL soon
- NodeSource gives you current LTS (v20) at `/usr/bin/node` — clean, simple, systemd-friendly

### 3. Create Swap Space (Required for Small VMs)

GCE VMs have **no swap by default**. On e2-micro (1GB RAM) or e2-small (2GB), `npm install` with native compilation can be OOM-killed.

```bash
# Check current swap (should show nothing)
sudo swapon --show

# Create 2GB swap
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make permanent (survives reboot)
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Tune swappiness (lower = less eager to swap, saves SSD writes)
sudo sysctl vm.swappiness=10
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf

# Verify
sudo swapon --show
```

### 4. Deploy the Agent

```bash
# Create a dedicated directory
sudo mkdir -p /opt/openagent
cd /opt/openagent

# Option A: Clone the repo and build
sudo git clone https://github.com/your-org/openagent-connect.git .
sudo npm install --production --workspace=@openagent/agent
sudo npm run build --workspace=@openagent/agent

# Option B: Copy pre-built files from your dev machine
# scp -r dist/ node_modules/ user@VM_IP:/opt/openagent/
```

If `npm install` is killed by OOM even with swap, build on a larger machine and copy the `dist/` and `node_modules/` directories.

### 5. Create a Dedicated System User

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin openagent
sudo chown -R openagent:openagent /opt/openagent
```

### 6. Create the systemd Service

Create `/etc/systemd/system/openagent-agent.service`:

```ini
[Unit]
Description=OpenAgent Connect Agent
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=openagent
Group=openagent
WorkingDirectory=/opt/openagent

ExecStart=/usr/bin/node /opt/openagent/apps/agent/dist/index.js \
  --server https://api.openclawinstall.net \
  --token REPLACE_TOKEN \
  --client REPLACE_CLIENT_ID

Restart=always
RestartSec=5
StartLimitIntervalSec=60
StartLimitBurst=5

# Environment
Environment=NODE_ENV=production

# Security hardening
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/opt/openagent

# Resource limits
LimitNOFILE=65536
MemoryMax=512M

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=openagent-agent

[Install]
WantedBy=multi-user.target
```

**For secrets**, use an environment file instead of putting the token in the unit file:

```bash
sudo mkdir -p /etc/openagent
sudo tee /etc/openagent/agent.env > /dev/null <<'EOF'
OPENAGENT_SERVER=https://api.openclawinstall.net
OPENAGENT_TOKEN=your-secret-token
OPENAGENT_CLIENT_ID=your-client-id
EOF
sudo chmod 600 /etc/openagent/agent.env
```

Then in the unit file, replace the `ExecStart` and add:
```ini
EnvironmentFile=/etc/openagent/agent.env
ExecStart=/usr/bin/node /opt/openagent/apps/agent/dist/index.js \
  --server ${OPENAGENT_SERVER} \
  --token ${OPENAGENT_TOKEN} \
  --client ${OPENAGENT_CLIENT_ID}
```

### 7. Enable and Start

```bash
sudo systemctl daemon-reload
sudo systemctl enable openagent-agent
sudo systemctl start openagent-agent

# Check status
sudo systemctl status openagent-agent

# View logs (follow mode)
sudo journalctl -u openagent-agent -f
```

## Troubleshooting

### npm install Killed (OOM)

**Symptom:** `npm install` runs for a while then prints `Killed` with no other error.

**Confirm it's OOM:**
```bash
dmesg | grep -i "oom\|killed" | tail -5
```

**Fix:** Create swap space (Step 3 above). Or increase VM size temporarily:
```bash
# From your local machine (not the VM)
gcloud compute instances stop <INSTANCE> --zone <ZONE>
gcloud compute instances set-machine-type <INSTANCE> --zone <ZONE> --machine-type e2-medium
gcloud compute instances start <INSTANCE> --zone <ZONE>
# Run npm install, then resize back down
```

### node-pty Build Errors

**Symptom:**
```
gyp ERR! build error
make: g++: No such file or directory
```

**Fix:** Install build tools (Step 2 above). Verify:
```bash
g++ --version    # Should show g++ 11+ (Ubuntu 22.04) or g++ 13+ (Ubuntu 24.04)
python3 --version
make --version
```

### node-pty PTY Spawn Fails Under systemd

**Symptom:** Agent starts but commands fail with: "Failed to open terminal: Could not open pty master device"

**Why:** The `ProtectSystem=strict` directive in the systemd unit may block access to `/dev/ptmx`.

**Fix:** Add these to the `[Service]` section:
```ini
DeviceAllow=/dev/ptmx rw
DeviceAllow=/dev/pts/* rw
```

Or change `ProtectSystem=strict` to `ProtectSystem=full` (less restrictive).

### SSH Disconnect Kills the Agent (Not Using systemd)

**Symptom:** You started the agent directly in SSH, disconnected, and the agent died.

**Why:** SSH sends SIGHUP to child processes on disconnect.

**Quick fix (testing only):**
```bash
# Option 1: nohup
nohup node /opt/openagent/apps/agent/dist/index.js \
  --server https://api.openclawinstall.net \
  --token TOKEN --client CLIENT_ID \
  > /var/log/openagent.log 2>&1 &

# Option 2: tmux
tmux new-session -d -s openagent \
  'node /opt/openagent/apps/agent/dist/index.js --server https://api.openclawinstall.net --token TOKEN --client CLIENT_ID'
# Reattach: tmux attach -t openagent

# Option 3: screen
screen -dmS openagent \
  node /opt/openagent/apps/agent/dist/index.js --server https://api.openclawinstall.net --token TOKEN --client CLIENT_ID
# Reattach: screen -r openagent
```

**Production fix:** Use the systemd service (Step 6 above). It survives SSH disconnects, reboots, and auto-restarts on crash.

### Firewall Blocks Outbound WSS

**Symptom:** Agent starts but can't connect. "Connector error: connect ETIMEDOUT".

**Default GCE behavior:** All outbound traffic is allowed. This only fails if:
1. A custom VPC egress deny rule exists
2. The VM has no external IP and no Cloud NAT

**Diagnose:**
```bash
# Test outbound HTTPS
curl -sS https://api.openclawinstall.net/health
# Should return: {"ok":true,...}

# If this fails, check VM's network config
gcloud compute instances describe <INSTANCE> --zone <ZONE> \
  --format='get(networkInterfaces[0].accessConfigs[0].natIP)'
# If empty, VM has no external IP
```

**Fix — add external IP:**
```bash
gcloud compute instances add-access-config <INSTANCE> --zone <ZONE>
```

**Fix — add Cloud NAT (if external IP is not desired):**
```bash
gcloud compute routers create nat-router --network=default --region=<REGION>
gcloud compute routers nats create nat-config \
  --router=nat-router --region=<REGION> \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges
```

### Node.js Too Old (Ubuntu 22.04 Default)

**Symptom:** `SyntaxError: Unexpected token '??='` or `SyntaxError: Unexpected token 'export'`

**Why:** You installed Node.js from Ubuntu's default repo (`apt install nodejs`) which gives v12 on 22.04.

**Fix:** Remove and reinstall via NodeSource:
```bash
sudo apt remove -y nodejs npm
sudo apt autoremove -y
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

### Disk Space Exhaustion

**Symptom:** Agent crashes, `npm install` fails, or logs stop writing. `df -h /` shows 0% free.

**Default GCE boot disk is 10GB.** Approximate usage:

| Component | Size |
|-----------|------|
| Ubuntu base OS | ~3.5 GB |
| Node.js + build-essential | ~500 MB |
| Agent + node_modules | ~15 MB |
| Swap file | 2 GB |
| Journal logs (growing) | up to 1 GB |

**Fix — use 20GB boot disk** when creating the VM:
```bash
gcloud compute instances create <NAME> --boot-disk-size=20GB ...
```

**Fix — limit journal size:**
```bash
sudo tee /etc/systemd/journald.conf.d/size.conf > /dev/null <<EOF
[Journal]
SystemMaxUse=200M
MaxRetentionSec=7day
EOF
sudo systemctl restart systemd-journald
```

**Fix — clean apt cache:**
```bash
sudo apt clean
```

### Unattended Upgrades Break Node.js

**Symptom:** Agent stops working after an automatic system update.

**Fix — pin Node.js version:**
```bash
sudo apt-mark hold nodejs
```

## GCE Startup Script (Fully Automated)

For automated VM provisioning, use a GCE startup script:

```bash
gcloud compute instances create openagent-vm \
  --zone=us-central1-a \
  --machine-type=e2-small \
  --image-family=ubuntu-2404-lts \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=20GB \
  --metadata=startup-script='#!/bin/bash
set -euo pipefail

# Swap
fallocate -l 2G /swapfile && chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile
echo "/swapfile none swap sw 0 0" >> /etc/fstab

# Node.js
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt-get install -y nodejs build-essential python3

# Agent
mkdir -p /opt/openagent
cd /opt/openagent
# ... deploy agent files ...
useradd --system --no-create-home --shell /usr/sbin/nologin openagent
chown -R openagent:openagent /opt/openagent

# systemd service (write unit file here)
systemctl daemon-reload
systemctl enable --now openagent-agent
'
```

## Verification

After the systemd service is running:

```bash
# Service status
sudo systemctl status openagent-agent

# Recent logs
sudo journalctl -u openagent-agent -n 50 --no-pager

# Process running
pgrep -f "node.*agent" && echo "Agent running" || echo "Agent NOT running"

# Outbound connectivity
curl -sS https://api.openclawinstall.net/health

# System resources
free -h           # RAM + swap usage
df -h /           # Disk usage
uptime            # Load average
```

## Quick Reference

```
Node.js:        /usr/bin/node (via NodeSource, NOT Ubuntu default)
Agent path:     /opt/openagent/
Service name:   openagent-agent
Logs:           journalctl -u openagent-agent -f
Start/stop:     systemctl start|stop|restart openagent-agent
Enable on boot: systemctl enable openagent-agent
Secrets:        /etc/openagent/agent.env (chmod 600)
Swap:           /swapfile (2GB)
```
