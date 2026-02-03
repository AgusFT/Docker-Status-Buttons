# Docker-Status-Buttons (GNOME + Ubuntu 24)

> **100% manual, transparent, audit-friendly guide** to build:
>
> 1) A **pill toggle** inside **GNOME Quick Settings** to **turn ALL Docker ON/OFF**:
>    - Host Docker Engine (`docker.service`, `docker.socket`, `containerd.service`)
>    - Docker Desktop (user service `docker-desktop.service`)
> 2) An **always-visible top bar indicator** using **Executor**:
>    - `🐳 DOCKER: ON` / `🐳 DOCKER: OFF`
>
> ✅ Philosophy: **full user control**, **no hidden magic**, **short auditable scripts**, **no auto-downloads**, **no `curl | bash`**.

---

## Table of contents

- [Philosophy](#philosophy)
- [What this solves and why](#what-this-solves-and-why)
- [Requirements](#requirements)
- [Tested on](#tested-on)
- [Repo structure](#repo-structure)
- [Step-by-step guide (manual)](#step-by-step-guide-manual)
  - [Step 0 — Preparation](#step-0--preparation)
  - [Step 1 — Create scripts](#step-1--create-scripts)
  - [Step 2 — Terminal sanity checks](#step-2--terminal-sanity-checks)
  - [Step 3 — Install GNOME extensions](#step-3--install-gnome-extensions)
  - [Step 4 — Configure the toggle](#step-4--configure-the-toggle)
  - [Step 5 — Configure the always-visible indicator](#step-5--configure-the-always-visible-indicator)
  - [Step 6 — Verification](#step-6--verification)
- [Security model](#security-model)
- [Uninstall](#uninstall)
- [Troubleshooting](#troubleshooting)

---

## Philosophy

This repo is designed so anyone can reproduce the result with **full control** and confidence:

- ✅ You create each file and paste its contents (no opaque automation)
- ✅ Scripts are short, readable, and easy to audit
- ✅ Nothing downloads code automatically
- ✅ No `curl | bash`
- ✅ Uses standard tools only: `systemctl`, `systemctl --user`, and `pkexec`

---

## What this solves and why

On Ubuntu you may have **two separate “Docker stacks”** running:

1) **Host Docker Engine** as systemd services:
   - `docker.service`
   - `docker.socket` (⚠️ if left enabled/active, systemd can auto-start Docker again)
   - `containerd.service`

2) **Docker Desktop (Linux)** as a user service (separate stack/VM):
   - `docker-desktop.service`

If you also use your PC for gaming, you want:
- Docker **OFF** while gaming (no background services consuming resources)
- Docker **ON** only when you need it for development

This setup adds:
- A GUI **toggle** to fully turn Docker ON/OFF (Quick Settings)
- An **always-visible status** to confirm whether anything Docker-related is still running

---

## Requirements

- Ubuntu 24.x with GNOME
- Docker Engine installed
- Docker Desktop installed (optional, but supported)
- Admin permissions to start/stop host services (Polkit prompt via `pkexec`)
- GNOME extensions:
  - **Custom Command Toggle** (pill toggle inside Quick Settings):  
    https://extensions.gnome.org/extension/7012/custom-command-toggle/
  - **Executor** (always-visible text on the top bar):  
    https://extensions.gnome.org/extension/2932/executor/

---

## Tested on

- Ubuntu 24.x + GNOME (e.g., GNOME 46)
- Works on both X11 and Wayland (systemd control is compositor-agnostic)
- Docker Engine + Docker Desktop installed (but you can use only one)

---

## Repo structure

This repo contains **reference templates**. The actual scripts are created in your HOME:

**Local files (your machine):**
- `~/.local/bin/docker-all-status`
- `~/.local/bin/docker-all-on`
- `~/.local/bin/docker-all-off`
- `~/.config/executor/docker-status.sh`

**Repo reference folders:**
- `scripts/` (templates)
- `executor/` (wrapper template)

---

# Step-by-step guide (manual)

## Step 0 — Preparation

To ensure Docker **does not auto-start**:

### 0.1 Disable host Docker Engine autostart
```bash
sudo systemctl disable docker docker.socket containerd
```

### 0.2 Disable Docker Desktop autostart (user service)
```bash
systemctl --user disable docker-desktop
```

### 0.3 (Optional) Stop everything right now
```bash
systemctl --user stop docker-desktop
sudo systemctl stop docker docker.socket containerd
```

---

## Step 1 — Create scripts

### 1.1 Create directories
```bash
mkdir -p ~/.local/bin
mkdir -p ~/.config/executor
```

---

### 1.2 Create `~/.local/bin/docker-all-status`

**What it does:** detects whether anything Docker-related is active (host Engine or Desktop) and prints:
- `🐳 DOCKER: ON`
- `🐳 DOCKER: OFF`

1) Open the file:
```bash
nano ~/.local/bin/docker-all-status
```

2) Paste:
```bash
#!/usr/bin/env bash
set -euo pipefail

# Detect whether host Docker Engine or Docker Desktop is active.
is_on=0

for unit in docker.service docker.socket containerd.service; do
  if /usr/bin/systemctl is-active --quiet "$unit"; then
    is_on=1
  fi
done

if /usr/bin/systemctl --user is-active --quiet docker-desktop.service 2>/dev/null; then
  is_on=1
fi

if [ "$is_on" -eq 1 ]; then
  echo "🐳 DOCKER: ON"
else
  echo "🐳 DOCKER: OFF"
fi
```

3) Save and exit (nano: `Ctrl+O`, Enter, `Ctrl+X`)

---

### 1.3 Create `~/.local/bin/docker-all-on`

**What it does:** starts the host Engine and Docker Desktop (if present).

1) Open:
```bash
nano ~/.local/bin/docker-all-on
```

2) Paste:
```bash
#!/usr/bin/env bash
set -euo pipefail

# Start host Docker Engine (requires Polkit authorization).
/usr/bin/pkexec /usr/bin/systemctl start containerd.service docker.socket docker.service

# Start Docker Desktop user service if it exists.
# If not installed, do nothing (no failure).
 /usr/bin/systemctl --user start docker-desktop.service 2>/dev/null || true
```

3) Save and exit.

---

### 1.4 Create `~/.local/bin/docker-all-off`

**What it does:** stops Docker Desktop first, then stops host Engine + socket + containerd.

⚠️ Important: we stop `docker.socket` too, because if it remains active, systemd can restart Docker when the socket is touched.

1) Open:
```bash
nano ~/.local/bin/docker-all-off
```

2) Paste:
```bash
#!/usr/bin/env bash
set -euo pipefail

# Stop Docker Desktop first (separate VM/stack).
/usr/bin/systemctl --user stop docker-desktop.service 2>/dev/null || true

# Stop host Docker Engine + socket + containerd.
# If docker.socket stays active, it can re-activate dockerd.
 /usr/bin/pkexec /usr/bin/systemctl stop docker.service docker.socket containerd.service
```

3) Save and exit.

---

### 1.5 Create Executor wrapper: `~/.config/executor/docker-status.sh`

**What it does:** Executor runs this wrapper and prints the status on the top bar.

1) Open:
```bash
nano ~/.config/executor/docker-status.sh
```

2) Paste:
```bash
#!/usr/bin/env bash
set -euo pipefail
"$HOME/.local/bin/docker-all-status"
```

3) Save and exit.

---

### 1.6 Make scripts executable
```bash
chmod +x ~/.local/bin/docker-all-status ~/.local/bin/docker-all-on ~/.local/bin/docker-all-off
chmod +x ~/.config/executor/docker-status.sh
```

---

## Step 2 — Terminal sanity checks

Status:
```bash
~/.local/bin/docker-all-status
```

Turn ON (you will be prompted for authorization):
```bash
~/.local/bin/docker-all-on
```

Verify:
```bash
~/.local/bin/docker-all-status
```

Turn OFF (you will be prompted for authorization):
```bash
~/.local/bin/docker-all-off
```

Verify:
```bash
~/.local/bin/docker-all-status
```

---

## Step 3 — Install GNOME extensions

### Option A (recommended): Extension Manager
Install:
```bash
sudo apt update
sudo apt install -y gnome-shell-extension-manager
```

Open **Extension Manager** and install:
- **Custom Command Toggle**
- **Executor**

### Option B: GNOME Extensions website
- Custom Command Toggle:
  https://extensions.gnome.org/extension/7012/custom-command-toggle/
- Executor:
  https://extensions.gnome.org/extension/2932/executor/

---

## Step 4 — Configure the toggle

Open preferences:
```bash
gnome-extensions prefs custom-command-toggle@storageb.github.com
```

Create a toggle (use absolute paths):

- **Button name:** `Docker-Status`

- **Toggle ON command:**
  - `/home/YOUR_USER/.local/bin/docker-all-on`

- **Toggle OFF command:**
  - `/home/YOUR_USER/.local/bin/docker-all-off`

- **Check Status command:**
  - `/home/YOUR_USER/.local/bin/docker-all-status`

- **Search term:** `ON`

Enable if available:
- “Keep toggle state synced”
- “Check command exit code”
- “Initial state from command output”

> Note: many GNOME extensions do not expand `~`, so use `/home/YOUR_USER/...`.

The toggle appears inside **Quick Settings** (top-right panel: Wi‑Fi/sound/battery).

---

## Step 5 — Configure the always-visible indicator

Open preferences:
```bash
gnome-extensions prefs executor@raujonas.github.io
```

In Executor, add a command that runs:
- `~/.config/executor/docker-status.sh`

Recommended:
- Interval: 5–10 seconds
- Position: Center (or your preference)

Expected result on the top bar:
- `🐳 DOCKER: OFF` / `🐳 DOCKER: ON`

---

## Step 6 — Verification

### 6.1 Host services
```bash
systemctl is-active docker.service docker.socket containerd.service
```
When OFF: should report `inactive`.

### 6.2 Desktop user service
```bash
systemctl --user is-active docker-desktop.service
```
When OFF: `inactive`.

### 6.3 Processes (OFF = should print nothing)
```bash
ps -eo pid,comm,args | grep -E "dockerd|containerd|docker-desktop|qemu|vpnkit" | grep -v grep
```

### 6.4 Docker CLI (OFF = should fail)
```bash
docker info
```

---

## Security model

**What it does:**
- Starts/stops services with `systemctl`
- Starts/stops user services with `systemctl --user`
- Uses `pkexec systemctl ...` for operations requiring admin privileges

**What it does NOT do:**
- No downloads
- No remote code execution
- No system file modifications
- No persistent background agents
- No Docker config changes (only ON/OFF)

**Why `pkexec`:**
- Standard Polkit mechanism to request explicit authorization securely.

---

## Uninstall

### 1) Stop everything
```bash
systemctl --user stop docker-desktop
sudo systemctl stop docker docker.socket containerd
```

### 2) Remove scripts and wrapper
```bash
rm -f ~/.local/bin/docker-all-status ~/.local/bin/docker-all-on ~/.local/bin/docker-all-off
rm -f ~/.config/executor/docker-status.sh
```

### 3) (Optional) Disable extensions
From Extension Manager or:
```bash
gnome-extensions disable custom-command-toggle@storageb.github.com
gnome-extensions disable executor@raujonas.github.io
```

### 4) (Optional) Re-enable autostart (restore default behavior)
```bash
sudo systemctl enable docker docker.socket containerd
systemctl --user enable docker-desktop
```

---

## Troubleshooting

### I don't see the toggle
- It lives inside **Quick Settings** (not permanently on the top bar).
- Try logout/login to refresh GNOME Shell.

### The toggle doesn't run anything
- Check permissions:
```bash
ls -la ~/.local/bin/docker-all-*
```
They must be executable: `-rwxr-xr-x`

- Use absolute paths in the toggle config (not `~`).

### Executor does not show the 🐳 emoji
Install emoji font:
```bash
sudo apt update
sudo apt install -y fonts-noto-color-emoji
```
Then logout/login.

### Docker turns back ON “by itself”
Make sure `docker.socket` is disabled:
```bash
systemctl is-enabled docker docker.socket containerd
```
All should be `disabled`.
