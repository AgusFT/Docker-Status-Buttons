# 🇺🇸 English (Full Guide)

> **Goal:** Full, transparent user control over Docker on Ubuntu 24 + GNOME.
>
> This guide helps you build:
> 1) A **Quick Settings pill toggle** to turn **ALL Docker ON/OFF**:
>    - Host Docker Engine (`docker.service`, `docker.socket`, `containerd.service`)
>    - Docker Desktop user service (`docker-desktop.service`)
> 2) An **always-visible top bar status** using **Executor**:
>    - `🐳 DOCKER: ON` / `🐳 DOCKER: OFF`

---

## Philosophy: full control, zero hidden magic

- ✅ Fully manual: you create files and paste content
- ✅ Short, auditable scripts
- ✅ No auto-downloads
- ✅ No `curl | bash`
- ✅ Only standard commands: `systemctl`, `systemctl --user`, `pkexec`

---

## Requirements

- Ubuntu 24.x with GNOME
- Docker Engine installed
- Docker Desktop installed (optional but supported)
- Admin rights (Polkit prompt via `pkexec`)
- GNOME extensions:
  - Custom Command Toggle:
    - https://extensions.gnome.org/extension/7012/custom-command-toggle/
  - Executor:
    - https://extensions.gnome.org/extension/2932/executor/

---

## Step 0 — Preparation (recommended)

Disable autostart so Docker won’t run unless you want it.

### 0.1 Disable host Docker Engine autostart
~~~bash
sudo systemctl disable docker docker.socket containerd
~~~

### 0.2 Disable Docker Desktop autostart (user service)
~~~bash
systemctl --user disable docker-desktop
~~~

### 0.3 (Optional) Stop everything now
~~~bash
systemctl --user stop docker-desktop
sudo systemctl stop docker docker.socket containerd
~~~

---

## Step 1 — Create scripts locally (manual)

### 1.1 Create directories
~~~bash
mkdir -p ~/.local/bin
mkdir -p ~/.config/executor
~~~

---

## Step 2 — Create files (one by one) and paste content

### 2.1 Create `~/.local/bin/docker-all-status`
~~~bash
nano ~/.local/bin/docker-all-status
~~~

Paste:
~~~bash
#!/usr/bin/env bash
set -euo pipefail

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
~~~

### 2.2 Create `~/.local/bin/docker-all-on`
~~~bash
nano ~/.local/bin/docker-all-on
~~~

Paste:
~~~bash
#!/usr/bin/env bash
set -euo pipefail

/usr/bin/pkexec /usr/bin/systemctl start containerd.service docker.socket docker.service
 /usr/bin/systemctl --user start docker-desktop.service 2>/dev/null || true
~~~

### 2.3 Create `~/.local/bin/docker-all-off`
~~~bash
nano ~/.local/bin/docker-all-off
~~~

Paste:
~~~bash
#!/usr/bin/env bash
set -euo pipefail

/usr/bin/systemctl --user stop docker-desktop.service 2>/dev/null || true
 /usr/bin/pkexec /usr/bin/systemctl stop docker.service docker.socket containerd.service
~~~

### 2.4 Create Executor wrapper `~/.config/executor/docker-status.sh`
~~~bash
nano ~/.config/executor/docker-status.sh
~~~

Paste:
~~~bash
#!/usr/bin/env bash
set -euo pipefail
"$HOME/.local/bin/docker-all-status"
~~~

### 2.5 Make executable
~~~bash
chmod +x ~/.local/bin/docker-all-status ~/.local/bin/docker-all-on ~/.local/bin/docker-all-off
chmod +x ~/.config/executor/docker-status.sh
~~~

---

## Step 3 — Terminal sanity checks
~~~bash
~/.local/bin/docker-all-status
~/.local/bin/docker-all-on
~/.local/bin/docker-all-status
~/.local/bin/docker-all-off
~/.local/bin/docker-all-status
~~~

---

## Step 4 — Install GNOME extensions (manual)

### Option A (recommended): Extension Manager
~~~bash
sudo apt update
sudo apt install -y gnome-shell-extension-manager
~~~

Install:
- Custom Command Toggle
- Executor

### Option B: GNOME Extensions website
Install from:
- https://extensions.gnome.org/extension/7012/custom-command-toggle/
- https://extensions.gnome.org/extension/2932/executor/

---

## Step 5 — Configure the pill toggle (Custom Command Toggle)

Open preferences:
~~~bash
gnome-extensions prefs custom-command-toggle@storageb.github.com
~~~

Create a toggle:
- **Button name:** `Docker-Status`
- **Toggle ON command (absolute path):**
  - `/home/YOUR_USER/.local/bin/docker-all-on`
- **Toggle OFF command (absolute path):**
  - `/home/YOUR_USER/.local/bin/docker-all-off`
- **Check Status command (absolute path):**
  - `/home/YOUR_USER/.local/bin/docker-all-status`
- **Search term:** `ON`

Enable if available:
- “Keep toggle state synced”
- “Check command exit code”
- “Initial state from command output”

---

## Step 6 — Always-visible status (Executor)

Open preferences:
~~~bash
gnome-extensions prefs executor@raujonas.github.io
~~~

Add command:
- `~/.config/executor/docker-status.sh`

Recommended:
- interval: 5–10s
- position: Center

---

## Verification (audit commands)

### Host services
~~~bash
systemctl is-active docker.service docker.socket containerd.service
~~~

### Desktop user service
~~~bash
systemctl --user is-active docker-desktop.service
~~~

### Processes (should be empty when OFF)
~~~bash
ps -eo pid,comm,args | grep -E "dockerd|containerd|docker-desktop|qemu|vpnkit" | grep -v grep
~~~

### CLI should fail when OFF
~~~bash
docker info
~~~
