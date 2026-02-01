# Lenovo ThinkPad Home Server Build Guide

> Repurposing a ~2010 Lenovo ThinkPad (likely T410) as a headless Linux server
> for learning local LLM infrastructure with Ollama.

---

## Progress Tracker

- [ ] **Phase 1: Hardware Audit** — Check specs, identify upgrades
- [ ] **Phase 2: Hardware Upgrades** — RAM and SSD if needed
- [ ] **Phase 3: Create Install Media** — Ubuntu Server USB from MacBook
- [ ] **Phase 4: Install Ubuntu Server** — Wipe Windows, install Linux
- [ ] **Phase 5: Base Server Setup** — Updates, SSH, networking
- [ ] **Phase 6: Install Ollama** — LLM runtime
- [ ] **Phase 7: Run a Model** — Pull and test a small LLM
- [ ] **Phase 8: Network Access** — Expose API to local network
- [ ] **Phase 9: Optional Web UI** — Browser-based chat interface

---

## Phase 1: Hardware Audit

**Goal:** Understand exactly what hardware you have before making any changes.
Run these on the Lenovo while it still has Windows.

### Check installed RAM (DIMM slots)

This tells you how many RAM sticks are installed, their size, and which
physical slots they occupy. You need this to know whether to buy 1 or 2
sticks when upgrading.

```cmd
wmic memorychip get BankLabel, Capacity, Speed, Manufacturer, DeviceLocator
```

- **Capacity** is in bytes. Divide by 1,073,741,824 to get GB.
- If you see **two rows**, both slots are populated.
- If you see **one row**, you have a free slot.
- The T410 supports a maximum of **8 GB** (2 x 4 GB DDR3 SO-DIMM).

### Check system summary

Gives a full overview: OS version, total RAM, processor, boot time, network
config. Useful as a before/after reference.

```cmd
systeminfo
```

### Confirm the exact laptop model

Important for looking up the hardware manual, compatible RAM, and BIOS
key bindings.

```cmd
wmic baseboard get Product, Manufacturer
```

### Check CPU details

Confirms core count and model. This determines what size LLM you can
realistically run (more cores = faster inference on CPU-only setups).

```cmd
wmic cpu get Name, NumberOfCores, NumberOfLogicalProcessors, MaxClockSpeed
```

### Check storage drive

Shows whether you have an HDD or SSD, and the drive size. If `MediaType`
says "Fixed hard disk media" with a large capacity, it is likely a
spinning HDD — worth replacing with an SSD.

```cmd
wmic diskdrive get Model, Size, MediaType
```

### Save results

Copy/paste all output into a text file for reference:

```cmd
systeminfo > C:\Users\%USERNAME%\Desktop\hardware_audit.txt
wmic memorychip get BankLabel, Capacity, Speed, Manufacturer, DeviceLocator >> C:\Users\%USERNAME%\Desktop\hardware_audit.txt
wmic cpu get Name, NumberOfCores, NumberOfLogicalProcessors, MaxClockSpeed >> C:\Users\%USERNAME%\Desktop\hardware_audit.txt
wmic diskdrive get Model, Size, MediaType >> C:\Users\%USERNAME%\Desktop\hardware_audit.txt
wmic baseboard get Product, Manufacturer >> C:\Users\%USERNAME%\Desktop\hardware_audit.txt
```

---

## Phase 2: Hardware Upgrades (If Needed)

**Goal:** Maximize RAM and use an SSD. These are the two biggest
performance factors for CPU-only LLM inference.

### RAM

- **Target:** 8 GB (2 x 4 GB DDR3 SO-DIMM, PC3-8500 or PC3-10600, 1.5V)
- **Why:** The LLM model gets loaded entirely into RAM. tinyllama needs
  ~1 GB, phi3:mini needs ~2-3 GB. The OS and Ollama runtime need another
  ~1-2 GB. At 4 GB total, you'll hit swap constantly (disk-based virtual
  memory), which destroys inference speed.
- **Cost:** ~$10-15 on eBay for a DDR3 SO-DIMM kit.
- **How:** Power off, remove battery, unscrew the RAM panel on the bottom
  of the laptop, insert/replace sticks. The ThinkPad hardware manual has
  diagrams.

### SSD

- **Target:** Any 2.5" SATA SSD, 120 GB minimum (256 GB is comfortable).
- **Why:** An SSD improves model load times (tinyllama loads in seconds
  vs. 30+ seconds on HDD), makes swap less painful, and speeds up every
  OS operation. If you're wiping the drive anyway, this is the time.
- **Cost:** ~$15-20 for a basic 240 GB SATA SSD.
- **How:** Power off, remove battery, unscrew the drive bay panel, swap
  the old drive into the new caddy or bracket.

---

## Phase 3: Create Install Media

**Goal:** Create a bootable Ubuntu Server USB drive from your MacBook.

### Why Ubuntu Server 24.04 LTS

- Long-term support through **2029** (vs. 2027 for 22.04)
- Newer kernel (6.8) has better hardware compatibility
- No GUI — less resource usage, everything runs via terminal/SSH
- Largest community and package ecosystem for server use

### Download the ISO

Download Ubuntu Server 24.04 LTS from:
https://ubuntu.com/download/server

The file will be named something like `ubuntu-24.04.x-live-server-amd64.iso`.

### Identify the USB drive (on MacBook)

Plug in your USB drive (8 GB minimum). This command lists all connected
disks so you can identify which one is the USB. **Getting this wrong will
erase the wrong drive.**

```bash
diskutil list
```

Look for your USB drive by its size. It will be something like `/dev/disk2`.
**Double-check the size matches your USB stick.**

### Unmount the USB drive

macOS auto-mounts USB drives. You need to unmount (not eject) before
writing to it. Replace `disk2` with your actual disk number.

```bash
diskutil unmountDisk /dev/disk2
```

### Write the ISO to the USB drive

`dd` copies raw data byte-by-byte onto the USB, making it bootable. The
`rdisk` variant is significantly faster than `disk` on macOS. **This
erases the USB drive completely.**

Replace `disk2` with your actual disk number. Replace the `.iso` filename
with the exact file you downloaded.

```bash
sudo dd if=~/Downloads/ubuntu-24.04.1-live-server-amd64.iso of=/dev/rdisk2 bs=4M status=progress
```

This may take a few minutes. When it finishes, macOS may show a "disk not
readable" popup — click **Eject**. That's normal.

### Alternative: Use Balena Etcher

If you're not comfortable with `dd`, download Balena Etcher
(https://etcher.balena.io). It provides a GUI: select the ISO, select the
USB, click Flash. Same result, less risk of picking the wrong disk.

---

## Phase 4: Install Ubuntu Server

**Goal:** Wipe Windows and install Ubuntu Server on the ThinkPad.

### Boot from USB

1. Plug the USB into the ThinkPad.
2. Power on and press **F12** repeatedly to open the boot menu.
   - If F12 doesn't work, try **F1** (BIOS setup) and change boot order.
3. Select the USB drive from the boot menu.

### Installation options

Walk through the Ubuntu installer with these choices:

| Setting | Choice | Why |
|---|---|---|
| Language | English | Default |
| Keyboard | Your layout | Match your physical keyboard |
| Install type | **Ubuntu Server** | No desktop environment, saves resources |
| Network | Use DHCP for now | You can set a static IP later |
| Proxy | Leave blank | Unless you have a corporate proxy |
| Mirror | Default | Auto-selected based on location |
| Storage | **Use entire disk** | Wipes Windows completely |
| LVM | Yes (default) | Allows flexible partition resizing later |
| Your name / server name / username | Your choice | The server name is its hostname on the network |
| Password | Set a strong password | This is your SSH and sudo password |
| **OpenSSH server** | **Install** | This is how you'll access the server remotely |
| Featured snaps | Skip all | Install things manually as needed |

Let the installer finish and **reboot**. Remove the USB when prompted.

### First login

The server boots to a text login prompt. Log in with the username and
password you set during install.

---

## Phase 5: Base Server Setup

**Goal:** Update the system, configure networking, and verify everything
works.

### Update all packages

Always do this first on a fresh install. `apt update` refreshes the
package index (what's available). `apt upgrade` installs newer versions
of installed packages.

```bash
sudo apt update && sudo apt upgrade -y
```

### Verify hardware is recognized

Check that the system sees the correct amount of RAM:

```bash
free -h
```

- **total** should match your installed RAM (minus a small amount reserved
  by the kernel).

Check CPU info:

```bash
lscpu
```

Check disk:

```bash
lsblk
```

### Find your IP address

You need this to SSH in from your MacBook.

```bash
ip addr show
```

Look for `inet` under your network interface (usually `enp0s25` or
`eth0` on ThinkPads). The address will look like `192.168.x.x`.

### Test SSH from your MacBook

From your MacBook's terminal, connect to the server. Replace the
username and IP with yours.

```bash
ssh your-username@192.168.x.x
```

If this works, you can close the ThinkPad lid and work entirely from
your MacBook from this point forward.

### (Optional) Set a static IP

DHCP can assign a different IP each time the server reboots. A static IP
means the address never changes, so you can always find your server.

Check your current network config:

```bash
ls /etc/netplan/
cat /etc/netplan/*.yaml
```

Edit the netplan config (filename may vary):

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Example static IP config (adjust to your network):

```yaml
network:
  version: 2
  ethernets:
    enp0s25:
      dhcp4: no
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

Apply the config:

```bash
sudo netplan apply
```

---

## Phase 6: Install Ollama

**Goal:** Install the Ollama runtime, which handles downloading, loading,
and serving LLM models.

### Why Ollama

- Single binary, no Python dependency hell
- Manages model downloads and quantization automatically
- Built-in REST API server
- Handles CPU-only inference out of the box
- Simplest path from zero to running an LLM

### Install

This script detects your OS and architecture, downloads the correct
binary, and sets up a systemd service so Ollama starts on boot.

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

### Verify it's running

Ollama runs as a background service. Check that it started successfully:

```bash
sudo systemctl status ollama
```

You should see `active (running)`. If not:

```bash
sudo systemctl start ollama
sudo systemctl enable ollama
```

### Test the API locally

Ollama listens on port 11434 by default. This should return a version
response:

```bash
curl http://localhost:11434/api/version
```

---

## Phase 7: Run a Model

**Goal:** Download and run a small LLM that fits in your available RAM.

### Model options for this hardware

| Model | RAM needed | Notes |
|---|---|---|
| **tinyllama** (1.1B) | ~1 GB | Smallest, fastest. Start here. |
| qwen2.5:1.5b | ~1.5 GB | Slightly better quality |
| phi3:mini (3.8B) | ~2.5 GB | Best quality that fits in 8 GB |

### Pull (download) a model

This downloads the model weights. Size varies by model (600 MB to 2+ GB).

```bash
ollama pull tinyllama
```

### Run interactively

This starts an interactive chat in your terminal. Type a message, get a
response. Type `/bye` to exit.

```bash
ollama run tinyllama
```

### Check performance

While the model is running, open another SSH session and monitor resource
usage:

```bash
# Real-time system monitor (CPU, RAM, processes)
htop

# Or simpler: check memory usage
free -h

# Check Ollama-specific process
ps aux | grep ollama
```

**Expected performance:** ~2-5 tokens/second on the i5-520M. That's
roughly one word per second. Slow but functional for learning.

---

## Phase 8: Network Access

**Goal:** Make Ollama accessible from other devices on your local network
(e.g., your MacBook).

### Why this is needed

By default, Ollama only listens on `localhost` (127.0.0.1), meaning only
the server itself can reach it. To query it from your MacBook, you need
to bind it to all network interfaces (`0.0.0.0`).

### Configure Ollama to listen on all interfaces

Edit the Ollama systemd service:

```bash
sudo systemctl edit ollama
```

This opens an override file. Add:

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
```

Save and exit, then restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

### Verify from your MacBook

Replace the IP with your server's address:

```bash
curl http://192.168.x.x:11434/api/version
```

### Send a chat request from your MacBook

```bash
curl http://192.168.x.x:11434/api/generate -d '{
  "model": "tinyllama",
  "prompt": "What is a home server?",
  "stream": false
}'
```

### (Optional) Open the firewall

Ubuntu may have `ufw` (Uncomplicated Firewall) active. If the curl from
your MacBook times out, allow the port:

```bash
sudo ufw allow 11434/tcp
sudo ufw status
```

---

## Phase 9: Optional Web UI

**Goal:** Add a browser-based chat interface so you don't need curl/terminal
to interact with the model.

### Open WebUI

Open WebUI is a self-hosted ChatGPT-style interface that connects to
Ollama's API.

### Install Docker (required for Open WebUI)

```bash
# Install Docker's official GPG key and repo
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Let your user run Docker without sudo
sudo usermod -aG docker $USER
```

Log out and back in for the group change to take effect:

```bash
exit
# SSH back in
```

### Run Open WebUI

```bash
docker run -d \
  --name open-webui \
  --network host \
  -v open-webui:/app/backend/data \
  -e OLLAMA_BASE_URL=http://127.0.0.1:11434 \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

Note: On this hardware, Docker + Open WebUI will use additional RAM. If
you're at 8 GB, this is fine. At 4 GB, skip this and use the REST API
directly.

### Access the UI

Open a browser on your MacBook and go to:

```
http://192.168.x.x:8080
```

Create an admin account on first visit. Select tinyllama as your model
and start chatting.

---

## Quick Reference

### Common commands

```bash
# Check server status
sudo systemctl status ollama

# List downloaded models
ollama list

# Pull a new model
ollama pull <model-name>

# Remove a model (free disk/RAM)
ollama rm <model-name>

# View system resources
htop
free -h
df -h

# Check Ollama logs
journalctl -u ollama -f

# Restart Ollama
sudo systemctl restart ollama

# Check your IP
ip addr show
```

### Useful API endpoints

```bash
# List models
curl http://localhost:11434/api/tags

# Generate a completion
curl http://localhost:11434/api/generate -d '{"model":"tinyllama","prompt":"hello","stream":false}'

# Chat (multi-turn)
curl http://localhost:11434/api/chat -d '{
  "model": "tinyllama",
  "messages": [{"role": "user", "content": "hello"}],
  "stream": false
}'
```

---

## Hardware Specs (Fill In After Phase 1)

| Component | Value |
|---|---|
| Laptop model | |
| CPU | |
| Cores / Threads | |
| RAM (total) | |
| RAM (slot 1) | |
| RAM (slot 2) | |
| Storage type | |
| Storage size | |
| Ubuntu version | |
| Server IP | |
