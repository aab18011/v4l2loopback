# v4l2loopback-multi (Debian 12)

> **Zero-delay camera switching in OBS** by prewiring IP/RTSP streams into multiple V4L2 dummy devices using a customized `v4l2loopback` module (>=16 devices), DKMS automation, and systemd services.

## Why this exists

OBS will often “handshake” and buffer when you switch to a new RTSP/HTTP source. That pause looks bad in live production. This project:

* Compiles a **custom v4l2loopback** with a higher device limit (>=16).
* Runs **ffmpeg** processes to **bridge IP cameras → `/dev/video*`** so OBS scenes can switch instantly.
* Uses **DKMS** so the module survives kernel updates.
* Provides **systemd units** to auto-build/update on boot (e.g., after **Wake-on-LAN**) and auto-launch per-camera ffmpeg feeds.
* Is designed to be **forked & redeployed** on future machines.

---

## Table of Contents

1. [Features](#features)
2. [Prerequisites](#prerequisites)
3. [Quick Start (TL;DR)](#quick-start-tldr)
4. [Build & Install the Patched Module (DKMS-safe)](#build--install-the-patched-module-dkms-safe)
5. [Autoload & Device Layout](#autoload--device-layout)
6. [Auto-Setup on Boot (WOL-friendly)](#auto-setup-on-boot-wol-friendly)
7. [Bridge IP Cameras → V4L2 with ffmpeg (systemd services)](#bridge-ip-cameras--v4l2-with-ffmpeg-systemd-services)
8. [OBS Configuration Notes](#obs-configuration-notes)
9. [Updating the Module](#updating-the-module)
10. [Troubleshooting & FAQ](#troubleshooting--faq)
11. [Uninstall / Rollback](#uninstall--rollback)
12. [Repository Layout](#repository-layout)
13. [Security & Secure Boot](#security--secure-boot)
14. [License & Credits](#license--credits)

---

## Features

* ✅ v4l2loopback compiled with **at least 16 devices**
* ✅ **DKMS** integration (auto-rebuild on kernel upgrades)
* ✅ **systemd** automation for boot/Wake-on-LAN
* ✅ **ffmpeg** templates for **low-latency RTSP → `/dev/videoX`**
* ✅ Clear **OBS** setup guidance
* ✅ **Troubleshooting** playbook and safe **uninstall** path

---

## Prerequisites

Debian 12 (Bookworm):

```bash
sudo apt update
sudo apt install -y \
  git build-essential linux-headers-$(uname -r) dkms \
  pkg-config \
  ffmpeg v4l-utils
```

> If you use Secure Boot, see [Security & Secure Boot](#security--secure-boot).

---

## Quick Start (TL;DR)

```bash
# 1) Clone your fork (recommended)
git clone https://github.com/<your-username>/v4l2loopback-multi.git
cd v4l2loopback-multi

# 2) Build & install current v4l2loopback with 16+ devices via DKMS-safe method
sudo ./scripts/setup_v4l2loopback.sh

# 3) Load with 16 devices and predictable labels/numbers
sudo modprobe v4l2loopback devices=16 exclusive_caps=1 \
  video_nr=0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15 \
  card_label='Cam0','Cam1','Cam2','Cam3','Cam4','Cam5','Cam6','Cam7','Cam8','Cam9','Cam10','Cam11','Cam12','Cam13','Cam14','Cam15'

# 4) Enable auto-load and ffmpeg bridges on boot
sudo ./scripts/install_systemd_units.sh
sudo systemctl enable --now v4l2loopback-setup.service
sudo systemctl enable --now ffmpeg@cam0.service  # repeat for each camera
```

---

## Build & Install the Patched Module (DKMS-safe)

This repo expects you to **fork** upstream and keep your **customization minimal and documented**. We raise the default device cap and provide automation.

### 1) Fork upstream and clone locally

* Upstream: `https://github.com/umlaeute/v4l2loopback`
* Fork to your account, then:

```bash
git clone https://github.com/<your-username>/v4l2loopback.git
cd v4l2loopback
```

### 2) Raise the device limit

Search and update `max_devices`:

```bash
grep -R "max_devices" -n
```

You should see a line similar to:

```c
static int max_devices = 8;
```

Change to:

```c
static int max_devices = 16; // or higher if you need
```

Commit the change to your fork.

### 3) Install via **DKMS-safe** method

> **Do not** run `dkms add .` directly inside the git folder (see DKMS note below). Use the copy-to-`/usr/src` method:

```bash
# In your customized v4l2loopback folder
VERSION=$(git describe --always --dirty 2>/dev/null || git describe --always 2>/dev/null || echo snapshot)
sudo cp -r . /usr/src/v4l2loopback-$VERSION
sudo dkms add v4l2loopback/$VERSION
sudo dkms build v4l2loopback/$VERSION
sudo dkms install v4l2loopback/$VERSION
```

---

## Autoload & Device Layout

Create **module autoload** and **options** so devices appear at boot with consistent numbering and labels:

```bash
# Ensure module loads at boot
echo v4l2loopback | sudo tee /etc/modules-load.d/v4l2loopback.conf

# Set options: 16 devices, fixed numbering, readable labels, OBS-friendly exclusive caps
sudo tee /etc/modprobe.d/v4l2loopback.conf >/dev/null <<'EOF'
options v4l2loopback devices=16 exclusive_caps=1 \
video_nr=0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15 \
card_label=Cam0,Cam1,Cam2,Cam3,Cam4,Cam5,Cam6,Cam7,Cam8,Cam9,Cam10,Cam11,Cam12,Cam13,Cam14,Cam15
EOF
```

Reload:

```bash
sudo modprobe -r v4l2loopback || true
sudo modprobe v4l2loopback
```

Verify:

```bash
lsmod | grep v4l2loopback
v4l2-ctl --list-devices
```

---

## Auto-Setup on Boot (WOL-friendly)

When this system wakes via **Wake-on-LAN**, we want it to:

1. Pull your fork (if present)
2. Build/install via DKMS-safe path
3. Load the module with your desired options

Add this **oneshot** service:

`/etc/systemd/system/v4l2loopback-setup.service`

```ini
[Unit]
Description=Build & load v4l2loopback (DKMS-safe) at boot
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/setup_v4l2loopback.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Install the script:

`/usr/local/bin/setup_v4l2loopback.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

# Where your fork lives for updates (optional)
REPO_DIR="/opt/v4l2loopback"
REPO_URL="https://github.com/<your-username>/v4l2loopback.git"

# Ensure deps
apt-get update
apt-get install -y git build-essential linux-headers-$(uname -r) dkms

# Get or update repo
if [ ! -d "$REPO_DIR/.git" ]; then
  git clone "$REPO_URL" "$REPO_DIR"
else
  git -C "$REPO_DIR" pull --ff-only
fi

cd "$REPO_DIR"

# Determine version string and copy to /usr/src
VERSION=$(git describe --always --dirty 2>/dev/null || git describe --always 2>/dev/null || echo snapshot)
SRC_DST="/usr/src/v4l2loopback-$VERSION"

# Clean old copy if mismatched
if [ -d "$SRC_DST" ]; then
  rm -rf "$SRC_DST"
fi
cp -r . "$SRC_DST"

# Remove older DKMS builds of the same module name to avoid conflicts
# (optional; safe because we're reinstalling right after)
dkms remove v4l2loopback/$VERSION --all >/dev/null 2>&1 || true

dkms add v4l2loopback/$VERSION
dkms build v4l2loopback/$VERSION
dkms install v4l2loopback/$VERSION

# Optionally handle Secure Boot signing here (see README section)
# e.g., mokutil --import /var/lib/dkms/mok.pub

# Load module with desired options
modprobe -r v4l2loopback 2>/dev/null || true
modprobe v4l2loopback devices=16 exclusive_caps=1 \
  video_nr=0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15 \
  card_label='Cam0','Cam1','Cam2','Cam3','Cam4','Cam5','Cam6','Cam7','Cam8','Cam9','Cam10','Cam11','Cam12','Cam13','Cam14','Cam15'
```

Make executable & enable:

```bash
sudo chmod +x /usr/local/bin/setup_v4l2loopback.sh
sudo systemctl enable --now v4l2loopback-setup.service
```

---

## Bridge IP Cameras → V4L2 with ffmpeg (systemd services)

We launch one **ffmpeg** process per camera to keep the streams “hot” and mapped to `/dev/videoX`.

### 1) Systemd template unit

`/etc/systemd/system/ffmpeg@.service`

```ini
[Unit]
Description=FFmpeg RTSP -> V4L2 bridge (%i)
After=network-online.target v4l2loopback-setup.service
Wants=network-online.target

[Service]
Type=simple
EnvironmentFile=-/etc/ffmpeg-cams/%i.env
# Minimal-latency options; adjust as needed for your cameras and network
ExecStart=/usr/bin/ffmpeg \
  -rtsp_transport tcp \
  -stimeout 5000000 \
  -reorder_queue_size 0 \
  -fflags nobuffer \
  -flags low_delay \
  -max_delay 0 \
  -analyzeduration 0 \
  -probesize 32k \
  -i ${RTSP_URL} \
  -vsync 1 -r ${OUT_FPS:-30} \
  -pix_fmt ${PIX_FMT:-yuv420p} \
  -vf scale=${SCALE:--2:1080} \
  -f v4l2 ${VIDEO_DEV}

Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

### 2) Per-camera environment files

Create a directory for camera definitions:

```bash
sudo mkdir -p /etc/ffmpeg-cams
```

Example **Cam0** → `/dev/video0`:

`/etc/ffmpeg-cams/cam0.env`

```bash
RTSP_URL="rtsp://user:pass@camera0.example:554/stream1"
VIDEO_DEV="/dev/video0"
OUT_FPS=30
PIX_FMT="yuv420p"
SCALE="-2:1080"  # auto width, 1080p height; keep aspect
```

Copy and adjust for `cam1.env`, `cam2.env`, etc. ensuring `VIDEO_DEV` matches your numbering and the labels you set.

### 3) Enable services

```bash
sudo systemctl enable --now ffmpeg@cam0.service
sudo systemctl enable --now ffmpeg@cam1.service
# ...repeat for each camera you want live
```

Check status/logs:

```bash
systemctl status ffmpeg@cam0
journalctl -u ffmpeg@cam0 -f
```

---

## OBS Configuration Notes

* Add **Video Capture Device** sources and select `/dev/video0`, `/dev/video1`, … (labels show as `Cam0`, `Cam1`, …).
* For **instant scene switching**, keep these sources **active** (don’t “deactivate when not shown” for critical feeds).
* If you see format mismatches, set the **Resolution/FPS Type** to **Custom** and match your ffmpeg output (e.g., 1920×1080 @ 30, NV12/YUYV/YUV420P).
* Consider **Buffering** disabled (or minimal) on the source to reduce delay.
* To avoid audio drift/conflicts, omit audio from these ffmpeg bridges (video only).

---

## Updating the Module

If your fork (or upstream) changes:

```bash
# If you used the systemd service, just reboot or run:
sudo systemctl start v4l2loopback-setup.service

# Manual:
cd /opt/v4l2loopback
git pull --ff-only
sudo /usr/local/bin/setup_v4l2loopback.sh
```

---

## Troubleshooting & FAQ

### Verify devices

```bash
v4l2-ctl --list-devices
ls -l /dev/video*
```

### Confirm module options

```bash
modinfo v4l2loopback | grep -E 'max_devices|exclusive_caps'
```

### Reduce latency further

Try tweaking ffmpeg flags: `-use_wallclock_as_timestamps 1`, remove `-r` to follow input fps, or use hardware decode if available.

### ffmpeg exits on RTSP dropout

systemd will auto-restart. Consider increasing `RestartSec` or using `-rw_timeout` (in µs) and camera keepalives.

### Devices appear but OBS can’t open them

Match pixel format and resolution; try `-pix_fmt yuv420p` or `-vf scale=1280:720`. Some capture stacks prefer `yuyv422`.

### DKMS says source path doesn’t exist

See the DKMS note below—use the copy-to-`/usr/src` approach or hardcode a `PACKAGE_VERSION` in `dkms.conf`.

---

## **Important DKMS Note (why `dkms add .` can fail)**

> You asked to include this verbatim guidance; here it is with headings for clarity.

Ah, yep — that DKMS error means the `dkms add .` step failed because the `dkms.conf` in `v4l2-loopback` is using a Makefile trick to auto-detect the version, and DKMS is choking on it.

The relevant part in the repo looks like:

```make
PACKAGE_VERSION := $(shell git describe --always --dirty 2>/dev/null || git describe --always 2>/dev/null || echo snapshot)
```

…and `dkms.conf` is using

```
PACKAGE_VERSION="$(PACKAGE_VERSION)"
```

When you run `dkms add .` directly in your working copy, DKMS tries to literally substitute that string without evaluating the `Makefile` logic — so you get

```
/usr/src/v4l2loopback-$(patsubst...) does not exist
```

---

### **Two ways to fix this**

#### **Option 1 — Let DKMS install from a clean `/usr/src` tree**

Instead of running `dkms add .` from the git folder, copy it into `/usr/src` with the version already substituted:

```bash
# Get actual version string
VERSION=$(git describe --always --dirty 2>/dev/null || git describe --always 2>/dev/null || echo snapshot)

# Copy source to /usr/src with the right folder name
sudo cp -r . /usr/src/v4l2loopback-$VERSION

# Add, build, install
sudo dkms add v4l2loopback/$VERSION
sudo dkms build v4l2loopback/$VERSION
sudo dkms install v4l2loopback/$VERSION
```

---

#### **Option 2 — Hardcode the version in dkms.conf**

Edit `dkms.conf` in your cloned repo and replace the `PACKAGE_VERSION=...` line with:

```bash
PACKAGE_VERSION="0.12.7-custom"
```

(or whatever your `git describe` shows, plus `-custom` if you want to distinguish it).

Then you can run:

```bash
sudo dkms add .
sudo dkms build v4l2loopback/0.12.7-custom
sudo dkms install v4l2loopback/0.12.7-custom
```

---

**Extra note:** That signing output means your kernel has **Secure Boot enabled**. If you want this module to load without disabling Secure Boot, you’ll have to enroll your signing key with `mokutil --import /var/lib/dkms/mok.pub` and reboot to complete MOK enrollment. Otherwise, you can temporarily disable Secure Boot in BIOS.

---

If you want, I can rewrite the auto-update script so it automatically:

* Detects the version
* Copies to `/usr/src` with the proper name
* Rebuilds with DKMS
* Handles Secure Boot signing

Do you want me to make that fixed script?

> **Note:** The provided `setup_v4l2loopback.sh` already implements Option 1 (copy to `/usr/src`), because it’s the most robust in automation.

---

## Uninstall / Rollback

List installed DKMS versions:

```bash
dkms status | grep v4l2loopback
```

Remove a specific version:

```bash
sudo modprobe -r v4l2loopback || true
sudo dkms remove v4l2loopback/<VERSION> --all
sudo rm -rf /usr/src/v4l2loopback-<VERSION>
```

Delete autoload and options (optional):

```bash
sudo rm -f /etc/modules-load.d/v4l2loopback.conf
sudo rm -f /etc/modprobe.d/v4l2loopback.conf
```

Disable ffmpeg services:

```bash
sudo systemctl disable --now ffmpeg@cam0.service ffmpeg@cam1.service
```

Remove the setup service/script if desired:

```bash
sudo systemctl disable --now v4l2loopback-setup.service
sudo rm -f /etc/systemd/system/v4l2loopback-setup.service
sudo rm -f /usr/local/bin/setup_v4l2loopback.sh
sudo systemctl daemon-reload
```

---

## Repository Layout

Recommended structure for your **portfolio-grade** fork:

```
.
├─ README.md
├─ scripts/
│  ├─ setup_v4l2loopback.sh             # DKMS-safe build & load
│  └─ install_systemd_units.sh          # Installs the units from this repo (optional helper)
├─ systemd/
│  ├─ v4l2loopback-setup.service
│  └─ ffmpeg@.service
├─ example-cams/
│  ├─ cam0.env
│  └─ cam1.env
└─ (upstream v4l2loopback source or submodule)
```

Example installer to drop units from your repo:

`scripts/install_systemd_units.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

cp -v systemd/v4l2loopback-setup.service /etc/systemd/system/
cp -v systemd/ffmpeg@.service /etc/systemd/system/

mkdir -p /etc/ffmpeg-cams
cp -n example-cams/*.env /etc/ffmpeg-cams/ 2>/dev/null || true

systemctl daemon-reload
echo "Systemd units installed. Edit /etc/ffmpeg-cams/*.env and enable services."
```

---

## Security & Secure Boot

* **Secure Boot enabled?**
  DKMS will sign modules with its own key; **you must enroll** that key via MOK:

  ```bash
  sudo mokutil --import /var/lib/dkms/mok.pub
  ```

  Reboot, follow the **MOK Manager** prompts, and complete enrollment.

* **Alternative:** disable Secure Boot in firmware/BIOS (less secure).

* **Least-privilege:** The ffmpeg services run as root by default. If you don’t need privileged ports or devices beyond `/dev/video*`, consider:

  * `User=obsbridge` + udev rules to grant group access to `/dev/video*`
  * `ProtectSystem=full`, `ProtectHome=true`, `PrivateTmp=true` in systemd units

---

## License & Credits

* **Upstream v4l2loopback**: © respective authors — see upstream license.
* **This repo’s scripts and docs**: MIT License
* **Aidan A. Bradley**: Fork maintainer

Thanks to the `v4l2loopback` maintainers and community, and the ffmpeg team.

---

### Final Notes

* You can raise `max_devices` beyond 16 if your workflow needs more.
* Consider adding CI (GitHub Actions) to lint scripts and validate the build on multiple kernels.
* For multi-host deployments, bake this into your provisioning (Ansible/Cloud-Init) with your fork URL pinned to a tag.
