

# v4l2loopback-multi (Debian 12)

> **Zero-delay camera switching in OBS** by prewiring IP/RTSP streams into multiple V4L2 dummy devices using a customized `v4l2loopback` module (>=16 devices), DKMS automation, and systemd services.

## Why this exists

OBS often buffers when switching to new RTSP/HTTP sources, causing pauses that disrupt live production. This project:

* Compiles a **custom v4l2loopback** with support for >=16 devices.
* Uses **ffmpeg** to bridge IP cameras to `/dev/video*` for instant OBS scene switching.
* Integrates **DKMS** to ensure module compatibility across kernel updates.
* Provides **systemd units** for automatic module building/loading and ffmpeg bridging on boot, including Wake-on-LAN scenarios.
* Is designed for **forking and redeployment** on new systems.

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

* ✅ v4l2loopback with **>=16 devices**
* ✅ **DKMS** integration for kernel upgrade compatibility
* ✅ **Systemd** automation for boot and Wake-on-LAN
* ✅ **ffmpeg** templates for low-latency RTSP → `/dev/videoX`
* ✅ Clear OBS setup instructions
* ✅ Robust troubleshooting and uninstall procedures

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

> For Secure Boot, see [Security & Secure Boot](#security--secure-boot).

---

## Quick Start (TL;DR)

```bash
# 1) Clone your fork (recommended)
git clone https://github.com/aab18011/v4l2loopback.git
cd v4l2loopback

# 2) Build & install v4l2loopback with 16+ devices via DKMS
sudo ./setup_v4l2loopback.sh

# 3) Enable auto-setup on boot
sudo cp setup_v4l2loopback.sh /usr/local/bin/
sudo chmod +x /usr/local/bin/setup_v4l2loopback.sh
sudo systemctl enable --now v4l2loopback-setup.service

# 4) Enable ffmpeg bridges for cameras
sudo mkdir -p /etc/ffmpeg-cams
# Create /etc/ffmpeg-cams/cam0.env, cam1.env, etc. (see below)
sudo systemctl enable --now ffmpeg@cam0.service  # repeat for each camera
```

---

## Build & Install the Patched Module (DKMS-safe)

Fork the upstream repo, customize the device limit, and install using DKMS.

### 1) Fork and clone

* Upstream: `https://github.com/umlaeute/v4l2loopback`
* Fork to your account, then:

```bash
git clone https://github.com/aab18011/v4l2loopback.git
cd v4l2loopback
```

### 2) Raise the device limit

Edit the source to increase `max_devices` (e.g., in `v4l2loopback.c`):

```c
static int max_devices = 16; // Changed from 8
```

Commit to your fork.

### 3) Install via DKMS

Run the provided script to handle DKMS-safe installation:

```bash
sudo ./setup_v4l2loopback.sh
```

This script:
* Checks for repo updates
* Cleans old/broken DKMS entries
* Installs the module for the current kernel
* Loads it with 16 devices

---

## Autoload & Device Layout

Configure the module to load at boot with consistent device numbering and labels:

```bash
# Ensure module loads at boot
echo v4l2loopback | sudo tee /etc/modules-load.d/v4l2loopback.conf

# Set options for 16 devices
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

The provided `setup_v4l2loopback.sh` ensures the module is built and loaded on boot or Wake-on-LAN.

### Service setup

`/etc/systemd/system/v4l2loopback-setup.service`:

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

Install and enable:

```bash
sudo cp setup_v4l2loopback.sh /usr/local/bin/
sudo chmod +x /usr/local/bin/setup_v4l2loopback.sh
sudo systemctl enable --now v4l2loopback-setup.service
```

The script checks for updates to your fork and rebuilds only if necessary, ensuring efficiency.

---

## Bridge IP Cameras → V4L2 with ffmpeg (systemd services)

Each camera gets an ffmpeg process to map RTSP streams to `/dev/videoX`.

### 1) Systemd template unit

`/etc/systemd/system/ffmpeg@.service`:

```ini
[Unit]
Description=FFmpeg RTSP -> V4L2 bridge (%i)
After=network-online.target v4l2loopback-setup.service
Wants=network-online.target

[Service]
Type=simple
EnvironmentFile=-/etc/ffmpeg-cams/%i.env
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

Create camera configs:

```bash
sudo mkdir -p /etc/ffmpeg-cams
```

Example `/etc/ffmpeg-cams/cam0.env`:

```bash
RTSP_URL="rtsp://user:pass@camera0.example:554/stream1"
VIDEO_DEV="/dev/video0"
OUT_FPS=30
PIX_FMT="yuv420p"
SCALE="-2:1080"
```

Repeat for `cam1.env`, `cam2.env`, etc., matching `VIDEO_DEV` to your device labels.

### 3) Enable services

```bash
sudo systemctl enable --now ffmpeg@cam0.service
sudo systemctl enable --now ffmpeg@cam1.service
# Repeat for each camera
```

Check status:

```bash
systemctl status ffmpeg@cam0
journalctl -u ffmpeg@cam0 -f
```

---

## OBS Configuration Notes

* Add **Video Capture Device** sources in OBS for `/dev/video0`, `/dev/video1`, etc. (labeled `Cam0`, `Cam1`, …).
* Keep sources **active** (avoid “deactivate when not shown”) for instant switching.
* Match resolution and format to ffmpeg output (e.g., 1920x1080 @ 30fps, YUV420P).
* Disable **Buffering** in OBS source settings for minimal latency.
* Exclude audio from ffmpeg bridges to avoid drift (use separate audio sources).

---

## Updating the Module

The `setup_v4l2loopback.sh` script checks for updates automatically:

```bash
sudo systemctl restart v4l2loopback-setup.service
```

Or manually:

```bash
cd /home/user/Documents/v4l2loopback
sudo ./setup_v4l2loopback.sh
```

The script only rebuilds if the git repo has new commits or the module isn’t installed for the current kernel.

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

### Reduce latency

Adjust ffmpeg flags: `-use_wallclock_as_timestamps 1`, remove `-r` to match input fps, or use hardware decoding if supported.

### ffmpeg exits on RTSP dropout

Systemd auto-restarts. Increase `RestartSec` or add `-rw_timeout` (in µs) for robustness.

### OBS can’t open devices

Ensure pixel format and resolution match. Try `-pix_fmt yuv420p` or `-vf scale=1280:720`. Some stacks prefer `yuyv422`.

### DKMS errors

The script uses a DKMS-safe copy-to-`/usr/src` approach to avoid version substitution issues. If errors persist, check `/var/log/dkms` logs.

---

## Uninstall / Rollback

List DKMS versions:

```bash
dkms status | grep v4l2loopback
```

Remove a version:

```bash
sudo modprobe -r v4l2loopback || true
sudo dkms remove v4l2loopback/<VERSION> --all
sudo rm -rf /usr/src/v4l2loopback-<VERSION>
sudo rm -rf /var/lib/dkms/v4l2loopback/<VERSION>
```

Remove configs:

```bash
sudo rm -f /etc/modules-load.d/v4l2loopback.conf
sudo rm -f /etc/modprobe.d/v4l2loopback.conf
```

Disable services:

```bash
sudo systemctl disable --now ffmpeg@cam0.service ffmpeg@cam1.service
sudo systemctl disable --now v4l2loopback-setup.service
sudo rm -f /etc/systemd/system/v4l2loopback-setup.service
sudo rm -f /usr/local/bin/setup_v4l2loopback.sh
sudo systemctl daemon-reload
```

---

## Repository Layout

```
.
├─ README.md
├─ setup_v4l2loopback.sh               # DKMS-safe build & load
├─ systemd/
│  ├─ v4l2loopback-setup.service
│  └─ ffmpeg@.service
├─ example-cams/
│  ├─ cam0.env
│  └─ cam1.env
└─ (v4l2loopback source)
```

Install systemd units:

```bash
sudo mkdir -p /etc/systemd/system
sudo cp systemd/* /etc/systemd/system/
sudo mkdir -p /etc/ffmpeg-cams
sudo cp example-cams/*.env /etc/ffmpeg-cams/
sudo systemctl daemon-reload
```

---

## Security & Secure Boot

* **Secure Boot**: Enroll the DKMS signing key:

```bash
sudo mokutil --import /var/lib/dkms/mok.pub
```

Reboot and follow MOK Manager prompts.

* **Alternative**: Disable Secure Boot in BIOS (less secure).

* **Least-privilege**:
  * Run ffmpeg services as a non-root user (e.g., `User=obsbridge`) with udev rules for `/dev/video*` access.
  * Add `ProtectSystem=full`, `ProtectHome=true`, `PrivateTmp=true` to systemd units.

---

## License & Credits

* **Upstream v4l2loopback**: © respective authors (see upstream license)
* **Scripts and docs**: MIT License
* **Maintainer**: Aidan A. Bradley

Thanks to the `v4l2loopback` and ffmpeg communities.

---

## Final Notes

* Increase `max_devices` beyond 16 if needed.
* Add CI (e.g., GitHub Actions) to validate builds.
* For multi-host setups, integrate with Ansible or Cloud-Init, pinning your fork’s URL to a tag.

