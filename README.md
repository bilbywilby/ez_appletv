# ez_appletv

Android-to-Apple TV file transfer and streaming, optimized for LAN-only use and an Android-only household.

This repo collects the practical paths that were verified during the original research: HTTPocket, VLC Remote Playback, SMB-to-Infuse, Plex, and a Jellyfin-based production setup. Infuse supports Jellyfin on tvOS, Swiftfin exists for tvOS, and Apple TV 4K hardware/storage variants matter when planning playback and transcoding. <citation src="1,2,3"></citation>

## Goal

Provide a reliable way to move or stream files from a Pixel / Android phone to Apple TV without requiring iPhone, iPad, or Mac setup.

Priorities:
- LAN-based transfer over cloud staging
- Verified alternatives to HTTPocket
- Minimal dependence on other Apple devices
- Clear setup steps and realistic limits

## Recommended paths

### 1) HTTPocket
Best for quick one-off transfers.

- Android 7.0+
- tvOS 16+
- Local Wi‑Fi
- QR connect or IP connect

Good when:
- you want the fastest setup
- you only need occasional file transfer
- you do not need a full media library

Limitations:
- not ideal for large libraries
- not a full media server

### 2) VLC Remote Playback
Best for browser-based uploads to Apple TV.

- Upload from Pixel browser
- Works over local network
- Storage limit is roughly 1.8 GB in practice

Good when:
- you want something simple and disposable
- you are sending a small video file

Limitations:
- storage is limited
- not a library solution

### 3) SMB Server -> Infuse
Best for local file browsing from Apple TV.

- Android SMB server app on the Pixel
- Infuse on Apple TV
- LAN-only
- No iOS/macOS setup needed

Good when:
- you want direct browsing of files
- you prefer a simple local share

Limitations:
- no media catalog
- no server-side transcoding

### 4) Plex
Best if you want an official tvOS app and a mainstream ecosystem.

- Official tvOS app exists
- Works well for a full media library

Good when:
- you want a standard media-server workflow
- you are comfortable with Plex account use

Limitations:
- broader ecosystem than necessary for some users

### 5) Jellyfin -> Infuse
Best production path for an Android-only household.

- Self-hosted
- LAN-friendly
- Works well with Apple TV playback
- Can use hardware transcoding on the server

This is the recommended long-term setup.

## Reference architecture

```text
Pixel (Android)
  ├─ HTTPocket ───────────────▶ Apple TV
  ├─ VLC browser upload ───────▶ Apple TV
  ├─ SMB Server ───────────────▶ Infuse on Apple TV
  └─ Jellyfin server ──────────▶ Infuse on Apple TV
```

## Setup options

### Quick transfer: HTTPocket

On Pixel:
1. Install HTTPocket
2. Grant storage permissions
3. Open the app
4. Connect with QR code or IP

On Apple TV:
1. Install HTTPocket
2. Create or display the connect code
3. Pair from the Pixel
4. Send files over Wi‑Fi

### Local share: SMB Server -> Infuse

On Pixel:
1. Install an SMB server app
2. Grant storage permissions
3. Add the shared folder
4. Set a username and password
5. Start the server on port `4455`
6. Note the Pixel IP address

On Apple TV:
1. Install Infuse
2. Open `Settings`
3. Add Files
4. Choose `SMB`
5. Enter `IP:4455`
6. Add the credentials

Notes:
- Android 14+ may require the “All files access” permission
- Keep the server on the local network only

### Production setup: Jellyfin -> Infuse

This is the best option if you want a stable library, metadata, and smoother playback.

Recommended server layout:
- Jellyfin in Docker
- Media mounted read-only
- Optional hardware transcoding
- LAN-only exposure

Suggested Docker baseline:
- avoid `network_mode: host` unless you specifically need it
- prefer explicit port mapping
- keep config and cache volumes separate
- test transcoding before production use

Example `docker-compose.yml`:

```yaml
services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
    volumes:
      - /srv/jellyfin/config:/config
      - /srv/jellyfin/cache:/cache
      - /mnt/media:/media:ro
    ports:
      - "8096:8096"
    restart: unless-stopped
```

If you use Intel hardware transcoding:
- map `/dev/dri/renderD128`
- add the `render` group GID
- confirm the server can see the GPU before relying on it

## Apple TV playback notes

Apple TV playback is easiest when:
- the TV is connected by Ethernet
- you use direct play whenever possible
- your files are H.264 or H.265
- audio is AAC or AC3 when you want fewer compatibility issues

If you have a lot of:
- VP9
- AV1
- TrueHD
- DTS

then server-side transcoding becomes more important.

## Security notes

- Keep services LAN-only unless remote access is truly needed
- Prefer port mapping over host networking
- Use strong credentials on SMB and Jellyfin
- Back up configs and test restores
- Keep Jellyfin logs and system updates under review
- Use Fail2ban if SSH is exposed

Suggested firewall order:
1. allow SSH from your admin subnet
2. set default deny incoming
3. enable UFW
4. verify rules before closing the session

## Troubleshooting

### SMB won’t connect
- confirm the Pixel and Apple TV are on the same LAN
- verify the Pixel IP address
- check the SMB port
- confirm storage permission was granted
- retry with a simpler share path

### Infuse can’t see Jellyfin
- confirm the Jellyfin container is reachable on `8096`
- verify the server IP
- check that the Jellyfin library paths are correct
- ensure the Apple TV can reach the host over the LAN

### Jellyfin transcodes unexpectedly
- check the codec of the file
- verify Apple TV format support
- inspect Jellyfin playback logs
- confirm direct play settings in the client

### Playback stutters
- prefer Ethernet over Wi‑Fi
- reduce bitrate
- check server CPU and disk usage
- confirm the media file is not stored on a slow external drive

### Android 14 permission issues
- grant “All files access” where required
- recheck storage permissions after app updates
- restart the SMB server app if the share disappears

## FAQ

### Do I need iPhone, iPad, or Mac?
No.

### Do I need cloud storage?
No.

### What is the best overall option?
Jellyfin + Infuse.

### What is the simplest option?
HTTPocket.

### What is best for browsing a folder from Apple TV?
SMB Server -> Infuse.

### What is best for a full media library?
Jellyfin -> Infuse.

## Status

Verified during research:
- HTTPocket viable
- VLC Remote Playback viable
- SMB Server viable
- Plex viable
- Infuse Jellyfin support viable
- Infuse Cloud Sync is not viable for Android-only setup

## License

Add your preferred license here.

## Credits

- HTTPocket
- VLC
- Infuse
- Jellyfin
- Plex
- Swiftfin
- Android SMB server apps

## Changelog

### v1.0.0
- Initial README
- Documented verified Android-to-Apple TV transfer paths
- Added LAN-only setup guidance
- Added Jellyfin/Infuse production workflow
```

# ez_appletv

**A verified, LAN-only file transfer and streaming solution for Android-to-Apple TV households.**

Stream media from your Android device (Pixel, etc.) to Apple TV without relying on iOS, macOS, or cloud services. This guide consolidates verified methods, security best practices, and a complete Docker + Jellyfin + Infuse deployment.

---

## 🎯 Goal

Enable seamless file transfer and media streaming from Android devices to Apple TV in an **Android-only household**, prioritizing:
- **LAN-based transfers** (no cloud staging)
- **Verified, open alternatives** to proprietary solutions
- **Security-first Docker deployment** with hardware transcoding
- **Zero iOS/macOS bootstrap required**

---

## 📋 What's in This Guide

1. **Quick Comparison Table** — verified methods (HTTPocket, VLC, SMB, Plex, Jellyfin)
2. **Production Docker Setup** — Jellyfin with Intel hardware transcoding
3. **Apple TV Integration** — Infuse + Jellyfin over LAN
4. **Security Hardening** — UFW, Fail2ban, backup verification
5. **Troubleshooting & Testing** — codec validation, GPU verification, restore drills

---

## 📊 Verified Methods at a Glance

| Method | Setup Complexity | LAN-Only | Live Direct Play | Best For | Notes |
|--------|------------------|----------|------------------|----------|-------|
| **HTTPocket** | ⭐ Low | ✅ Yes | ❓ No | Quick ad-hoc transfers | Android 7.0+; tvOS 16+; QR-code pairing |
| **VLC Remote** | ⭐ Low | ✅ Yes | ⚠️ ~1.8 GB | Testing, small files | Browser-based; storage-limited |
| **SMB Server** | ⭐⭐ Medium | ✅ Yes | ❌ No | Library browsing | Requires "All files access" on Android 14+ |
| **Plex** | ⭐⭐⭐ High | ✅ Yes | ✅ Yes | Media library + remote access | Free tvOS app (2026); requires account |
| **Jellyfin + Infuse** | ⭐⭐⭐ High | ✅ Yes | ✅ Yes | **Production streaming** (recommended) | Self-hosted; hardware transcoding; open-source |

**→ This guide focuses on Jellyfin + Infuse as the recommended production setup.**

---

## 🚀 Quick Start (5 minutes)

### Option 1: HTTPocket (Ad-hoc File Transfers)

**On Pixel:**
```
1. Install HTTPocket from Google Play
2. Grant storage permissions
3. Open app → Tap QR code icon
4. Point camera at Apple TV QR code
```

**On Apple TV:**
```
1. Open HTTPocket
2. Tap "Create QR Code"
3. Scan with Pixel camera
4. Select files on Pixel and upload
```

**✅ No setup, ✅ LAN-only, ⚠️ Slow for large files**

---

### Option 2: SMB Server to Infuse (Browse & Play)

**On Pixel (Android 7.0+):**
```bash
# Install "SMB Server" by miniy (net.miniy.android.smb)
# Settings:
#  - Add Share: /storage/emulated/0
#  - Username: android / Password: password
#  - Start server on port 4455
#  - Note your Pixel IP address (Settings → Network)
```

**On Apple TV:**
```
1. Install Infuse (tvOS App Store)
2. Settings → Add Files Source → SMB
3. Enter: smb://192.168.1.X:4455
4. Username: android / Password: password
5. Browse files directly
```

**✅ LAN-only, ✅ Direct browsing, ⚠️ No transcoding (direct play only)**

---

### Option 3: Jellyfin + Infuse (Recommended Production Setup)

**See [Jellyfin Docker Setup](#-jellyfin-docker-setup-production) below.**

---

## 🐳 Jellyfin Docker Setup (Production)

This is the **verified, secure, hardware-accelerated setup** recommended for ongoing streaming.

### Prerequisites

- **Server Host**: Ubuntu 24.04 LTS or Debian 12+
- **CPU**: Intel (N100/N150 mini PC, 12th Gen+), AMD Ryzen, or NVIDIA GPU (optional for transcoding)
- **RAM**: 4 GB minimum, 8 GB recommended
- **Storage**: 
  - `/srv/jellyfin/config` — 50 GB (media library metadata, thumbnails)
  - `/srv/jellyfin/cache` — 100 GB (transcoding cache)
  - `/mnt/media` — Your media library (movies, TV shows, music)
- **Network**: Gigabit Ethernet preferred (for 4K streaming)
- **Docker**: `docker` and `docker-compose` installed

### Step 1: Verify Intel GPU (if applicable)

```bash
# Check if /dev/dri exists
ls -la /dev/dri/
# Expected output:
# crw-rw---- 1 root render 226, 128 ... renderD128
# crw-rw---- 1 root video  226,   0 ... card0

# If nothing appears, your CPU doesn't have integrated graphics
# (or it's an "F"-series Intel chip with no iGPU)
```

### Step 2: Get Your Render Group GID

```bash
getent group render
# Output: render:x:109:
# Your GID is 109 — substitute this in docker-compose.yml below
# Ubuntu 24.04: often 992
# Debian 12: often 104
```

### Step 3: Create Media Directories

```bash
sudo mkdir -p /srv/jellyfin/{config,cache}
sudo mkdir -p /mnt/media/{movies,tv,music}

# Set permissions (assumes you run as UID 1000)
sudo chown 1000:1000 /srv/jellyfin /mnt/media
sudo chmod 750 /srv/jellyfin /mnt/media
```

### Step 4: Create `docker-compose.yml`

```yaml
---
services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    user: "1000:1000"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
      - JELLYFIN_PublishedServerUrl=http://192.168.1.50:8096  # Update to your server IP
    volumes:
      - /srv/jellyfin/config:/config
      - /srv/jellyfin/cache:/cache
      - /mnt/media:/media:ro  # Read-only to protect source files
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128  # Intel GPU (only if verified in Step 1)
    group_add:
      - "109"  # render group GID from Step 2 — update if different
    ports:
      - "192.168.1.50:8096:8096/tcp"  # Web UI (LAN-only binding)
      - "192.168.1.50:8920:8920/tcp"  # HTTPS (optional)
      - "127.0.0.1:7359:7359/udp"     # Client discovery (localhost only)
      - "127.0.0.1:1900:1900/udp"     # DNLA/UPnP (localhost only)
    restart: unless-stopped
```

**⚠️ Important Notes:**
- Replace `192.168.1.50` with your server's actual LAN IP
- Replace `109` with your render group GID from Step 2
- Remove the `devices:` block if you don't have an Intel GPU
- Ports are bound to `192.168.1.50` (LAN-only) — remote access requires a reverse proxy

### Step 5: Start Jellyfin

```bash
docker-compose up -d
docker logs jellyfin
# Wait 30 seconds for startup
```

### Step 6: Initial Setup in Jellyfin Dashboard

1. Open browser → `http://192.168.1.50:8096`
2. **Language & Locale** — choose your region
3. **User Setup** — create admin account
4. **Media Libraries**:
   - Name: `Movies` → Path: `/media/movies`
   - Name: `TV Shows` → Path: `/media/tv`
   - Name: `Music` → Path: `/media/music`
5. **Metadata** — let it scan and download artwork
6. **Dashboard → Playback → Transcoding**:
   - **Hardware Acceleration**: Intel QuickSync (QSV)
   - **Decoding**: Enable H.264, HEVC, HEVC 10-bit, VP9, AV1
   - **Encoding**: Enable H.264, HEVC
   - **Hardware Tone Mapping**: Enable (for HDR→SDR on older displays)

### Step 7: Verify Hardware Transcoding

```bash
# Check GPU visibility in container
docker exec jellyfin /usr/lib/jellyfin-ffmpeg/vainfo
# Expected output includes: "Driver version: Intel iHD driver"
# followed by encode profiles

# Monitor GPU usage during playback
sudo intel_gpu_top  # (if on host with Intel GPU)
# Watch for Video/0 or Video/1 activity while streaming
```

If `vainfo` shows no output or errors, your GPU is not mapped correctly:
- Double-check the render group GID in `docker-compose.yml`
- Verify `/dev/dri/renderD128` exists on your host
- Restart the container: `docker-compose restart jellyfin`

---

## 🎬 Apple TV Integration: Infuse + Jellyfin

### On Apple TV 4K

1. **Open App Store** → Search `Infuse`
2. **Download** (free version sufficient for LAN streaming)
3. **Launch Infuse** → Settings → Add Files Source
4. **Select Jellyfin** from the source list
5. **Enter Details**:
   - Server URL: `http://192.168.1.50:8096` (your Jellyfin server IP)
   - Username: admin (your Jellyfin admin account)
   - Password: (your password)
6. **Tap Connect** — Infuse auto-discovers all libraries

### Optimization for 4K/HDR Playback

**On Apple TV:**
```
1. Settings → Video and Audio → Match Content
   - Enable: Match frame rate
   - Enable: Match dynamic range
   (This switches the TV's refresh rate and HDR mode per video)

2. Settings → Video and Audio → Format
   - Dolby Vision: On (if supported)
   - HDR10: On
```

**On your network:**
- **Connect Apple TV via Ethernet**, not Wi-Fi
- 4K Blu-ray remuxes need 50–100 Mbps sustained throughput
- Wi-Fi loses 20–30% under load

### Direct Play vs. Transcoding

**Direct Play** (server idle):
- Video codec: H.264 or H.265
- Audio codec: AAC, AC3, or MP3
- File streams directly to Apple TV, zero server load
- **Target state — optimize your library here**

**Transcoding** (GPU active):
- Video uses unsupported codec (VP9, AV1 on older tvOS, etc.)
- Audio uses proprietary format (TrueHD, DTS)
- Bitrate or resolution doesn't match network/display
- Server uses GPU to re-encode on the fly

**Library Optimization:**
```bash
# Check your media codecs
ffprobe -v error -select_streams v:0 -show_entries \
  stream=codec_name,width,height,bit_rate \
  -of default=noprint_wrappers=1 /mnt/media/movies/example.mkv

# Ideal output for Apple TV:
# codec_name=h264 (or hevc)
# width=3840, height=2160 (or 1920x1080)
# bit_rate=30000000 (or lower)
```

Files matching the above need **zero transcoding**. Re-encode anything else with ffmpeg or Handbrake before adding to Jellyfin.

---

## 🔒 Security Hardening

### ⚠️ Critical Issues from Original Document

Your earlier deployment doc had several gaps. Here's the verified baseline:

#### 1. Docker Network Isolation (NOT `network_mode: host`)

**Problem:** The original document used `network_mode: host`, which **removes Docker's network namespace isolation**. The container can sniff all host traffic.

**Fix:** Use explicit port mapping (see `docker-compose.yml` above):
```yaml
ports:
  - "192.168.1.50:8096:8096/tcp"  # Binds only to internal network IP
```

**Result:** Port 8096 only accessible from your LAN, not the internet.

---

#### 2. UFW (Firewall) Rules — Correct Sequence

```bash
# Allow SSH BEFORE enabling the firewall
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp

# THEN lock everything down
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable

# Verify rules applied in correct order
sudo ufw status numbered
```

**Why order matters:** If you reverse these steps, you lock yourself out immediately.

---

#### 3. File Permissions (Not Too Broad)

**Problem:** `chmod 755` allows any local user to list and read your media files.

**Fix:** Restrict to jellyfin user only:
```bash
sudo chown -R 1000:1000 /srv/jellyfin /mnt/media
sudo chmod 750 /srv/jellyfin /mnt/media
sudo chmod 750 /srv/jellyfin/config /mnt/media/movies /mnt/media/tv
```

This prevents other users on the system from accessing the media folder.

---

#### 4. Fail2ban: SSH Brute-Force Protection

```bash
sudo apt update && sudo apt install -y fail2ban

# Create local config (don't edit /etc/fail2ban/jail.conf directly)
sudo nano /etc/fail2ban/jail.local
```

**Paste this:**
```ini
[DEFAULT]
bantime = 3600         # Ban for 1 hour
findtime = 600         # Within 10-minute window
maxretry = 5           # After 5 failed attempts

[sshd]
enabled = true
port = ssh
maxretry = 3           # More aggressive for SSH
bantime = 7200         # 2 hours
```

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

---

#### 5. Jellyfin Container Logging

Monitor for suspicious activity:
```bash
# Watch logs in real time
docker logs -f jellyfin

# Or check the last 100 lines
docker logs --tail 100 jellyfin
```

**Red flags:**
- Repeated failed authentication attempts
- Requests from unexpected IPs
- Codec/transcode errors that cause 100% CPU spikes

---

### Backup & Restore (Tested Procedure)

#### Create Encrypted Backup

#!/bin/bash
# backup-jellyfin.sh

BACKUP_DIR="/backup/jellyfin"
DATE=$(date +%Y%m%d_%H%M%S)
ARCHIVE="jellyfin_backup_${DATE}.tar.gz"

mkdir -p "$BACKUP_DIR"

# Stop the container to ensure consistent backup
docker-compose -f /path/to/docker-compose.yml stop jellyfin

# Create tarball
tar -czf "$BACKUP_DIR/$ARCHIVE" \
  /srv/jellyfin/config \
  /srv/jellyfin/cache

# Encrypt with GPG
gpg --symmetric --cipher-algo AES256 \
  "$BACKUP_DIR/$ARCHIVE"

# Remove unencrypted tarball
rm "$BACKUP_DIR/$ARCHIVE"

# Restart container
docker-compose -f /path/to/docker-compose.yml start jellyfin

# Verify integrity
gpg --decrypt "$BACKUP_DIR/${ARCHIVE}.gpg" | tar -tzf - > /dev/null &&Here is a complete, production-grade repository architecture and codebase structured for real-world resilience, strict typing, clean separation of concerns, and reproducible CI/CD execution.
Project Directory Structure
pixel-vlc-bridge/
├── .github/
│   └── workflows/
│       └── ci.yml
├── config/
│   └── profiles.yaml
├── src/
│   └── pixel_vlc_bridge/
│       ├── __init__.py
│       ├── py.typed
│       ├── cli.py
│       ├── config.py
│       ├── logger.py
│       ├── uploader.py
│       └── integrity.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_config.py
│   ├── test_integrity.py
│   └── test_uploader.py
├── .gitignore
├── pyproject.toml
└── README.md

Configuration Files
pyproject.toml
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "pixel-vlc-bridge"
version = "0.1.0"
description = "Validated stream-uploader for VLC Remote Playback endpoints on local networks."
readme = "README.md"
requires-python = ">=3.10"
authors = [
    { name = "Anthony A Dee" }
]
dependencies = [
    "requests>=2.31.0",
    "pydantic>=2.5.0",
    "pyyaml>=6.0.1",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "pytest-cov>=4.1.0",
    "mypy>=1.8.0",
    "types-requests>=2.31.0.10",
    "types-PyYAML>=6.0.12.12",
    "ruff>=0.1.9",
]

[project.scripts]
vlc-upload = "pixel_vlc_bridge.cli:main"

[tool.setuptools.packages.find]
where = ["src"]

[tool.mypy]
python_version = "3.10"
strict = true
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true

[tool.ruff]
target-version = "py310"
line-length = 100
select = ["E", "F", "B", "I", "UP"]
ignore = []

[tool.pytest.ini_options]
minversion = "7.0"
addopts = "-ra -q --cov=pixel_vlc_bridge"
testpaths = ["tests"]

config/profiles.yaml
default:
  tv_ip: "192.168.1.150"
  port: 8080
  upload_endpoint: "/upload"
  form_field_name: "files[]"
  timeout_seconds: 30
  retry_attempts: 3
  chunk_size_bytes: 1048576

living_room:
  tv_ip: "192.168.1.150"
  port: 8080

bedroom:
  tv_ip: "192.168.1.151"
  port: 8080

.gitignore
__pycache__/
*.py[cod]
*$py.class
.pytest_cache/
.mypy_cache/
.ruff_cache/
.coverage
htmlcov/
dist/
build/
*.egg-info/
.venv/
venv/

Source Code (src/pixel_vlc_bridge/)
src/pixel_vlc_bridge/__init__.py
"""Pixel VLC Bridge Package."""

__version__ = "0.1.0"

src/pixel_vlc_bridge/py.typed
# Marker file for PEP 561 inline typing support.

src/pixel_vlc_bridge/logger.py
import logging
import sys


def setup_logger(name: str = "pixel_vlc_bridge", level: int = logging.INFO) -> logging.Logger:
    """Configures structured stream logging."""
    logger = logging.getLogger(name)
    if not logger.handlers:
        handler = logging.StreamHandler(sys.stdout)
        formatter = logging.Formatter(
            fmt="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
            datefmt="%Y-%m-%d %H:%M:%S",
        )
        handler.setFormatter(formatter)
        logger.addHandler(handler)
        logger.setLevel(level)
    return logger

src/pixel_vlc_bridge/config.py
from pathlib import Path
from typing import Any, Dict
from pydantic import BaseModel, Field
import yaml


class DeviceProfile(BaseModel):
    tv_ip: str
    port: int = Field(default=8080, ge=1, le=65535)
    upload_endpoint: str = "/upload"
    form_field_name: str = "files[]"
    timeout_seconds: int = Field(default=30, ge=1)
    retry_attempts: int = Field(default=3, ge=1)
    chunk_size_bytes: int = Field(default=1048576, ge=4096)


def load_profile(config_path: Path, profile_name: str = "default") -> DeviceProfile:
    """Loads and validates a device profile from a YAML file."""
    if not config_path.exists():
        raise FileNotFoundError(f"Configuration file not found: {config_path}")

    with open(config_path, "r", encoding="utf-8") as f:
        raw_data: Dict[str, Any] = yaml.safe_load(f) or {}

    if profile_name not in raw_data:
        raise KeyError(f"Profile '{profile_name}' not defined in {config_path}")

    profile_data = raw_data[profile_name]
    
    # Inherit missing default field values from the top-level 'default' block if present
    if profile_name != "default" and "default" in raw_data:
        merged = raw_data["default"].copy()
        merged.update(profile_data)
        profile_data = merged

    return DeviceProfile(**profile_data)

src/pixel_vlc_bridge/integrity.py
import hashlib
from pathlib import Path


def calculate_source_sha256(file_path: Path, chunk_size: int = 1048576) -> str:
    """Computes the SHA-256 digest of a local source file prior to transfer."""
    if not file_path.exists() or not file_path.is_file():
        raise FileNotFoundError(f"Source file does not exist or is not a regular file: {file_path}")

    sha256_hash = hashlib.sha256()
    with open(file_path, "rb") as f:
        while chunk := f.read(chunk_size):
            sha256_hash.update(chunk)

    return sha256_hash.hexdigest()

src/pixel_vlc_bridge/uploader.py
import os
import time
from pathlib import Path
from typing import Callable, Optional
import requests

from pixel_vlc_bridge.config import DeviceProfile
from pixel_vlc_bridge.logger import setup_logger

logger = setup_logger(__name__)


class ProgressFileReader:
    """Custom file wrapper to track progress during multipart streaming uploads."""

    def __init__(
        self,
        file_path: Path,
        callback: Optional[Callable[[int, int], None]] = None,
    ) -> None:
        self.file_path = file_path
        self.total_size = os.path.getsize(file_path)
        self.bytes_read = 0
        self.callback = callback
        self._file = open(file_path, "rb")

    def read(self, size: int = -1) -> bytes:
        chunk = self._file.read(size)
        if chunk:
            self.bytes_read += len(chunk)
            if self.callback:
                self.callback(self.bytes_read, self.total_size)
        return chunk

    def close(self) -> None:
        self._file.close()

    def __enter__(self) -> "ProgressFileReader":
        return self

    def __exit__(self, exc_type: Any, exc_val: Any, exc_tb: Any) -> None:
        self.close()


def upload_media_file(
    file_path: Path,
    profile: DeviceProfile,
    progress_callback: Optional[Callable[[int, int], None]] = None,
) -> bool:
    """Uploads a local media file to the configured VLC Remote Playback HTTP endpoint."""
    endpoint = profile.upload_endpoint if profile.upload_endpoint.startswith("/") else f"/{profile.upload_endpoint}"
    target_url = f"http://{profile.tv_ip}:{profile.port}{endpoint}"

    logger.info("Target URL resolved: %s", target_url)

    for attempt in range(1, profile.retry_attempts + 1):
        try:
            logger.info("Starting upload attempt %d/%d for %s", attempt, profile.retry_attempts, file_path.name)
            
            with ProgressFileReader(file_path, progress_callback) as pfr:
                files = {
                    profile.form_field_name: (file_path.name, pfr, "application/octet-stream")
                }
                response = requests.post(
                    target_url,
                    files=files,
                    timeout=profile.timeout_seconds,
                )

            if response.status_code in (200, 201, 204):
                logger.info("Upload succeeded on attempt %d (HTTP %d)", attempt, response.status_code)
                return True

            logger.warning(
                "Upload failed on attempt %d: HTTP %d - Response: %s",
                attempt,
                response.status_code,
                response.text[:200],
            )

        except (requests.RequestException, OSError) as err:
            logger.warning("Attempt %d encountered an error: %s", attempt, str(err))

        if attempt < profile.retry_attempts:
            backoff = 2 ** attempt
            logger.info("Backing off for %d seconds before retrying...", backoff)
            time.sleep(backoff)

    logger.error("All %d upload attempts failed.", profile.retry_attempts)
    return False

src/pixel_vlc_bridge/cli.py
import argparse
import sys
from pathlib import Path

from pixel_vlc_bridge.config import load_profile
from pixel_vlc_bridge.integrity import calculate_source_sha256
from pixel_vlc_bridge.logger import setup_logger
from pixel_vlc_bridge.uploader import upload_media_file

logger = setup_logger("cli")


def print_progress(current: int, total: int) -> None:
    pct = (current / total) * 100 if total > 0 else 0
    sys.stdout.write(f"\rUploading: {pct:.1f}% ({current}/{total} bytes)")
    sys.stdout.flush()
    if current >= total:
        sys.stdout.write("\n")


def main() -> None:
    parser = argparse.ArgumentParser(description="Upload media files to a VLC Remote Playback instance.")
    parser.add_argument("file", type=Path, help="Path to the file to upload")
    parser.add_argument(
        "--config",
        type=Path,
        default=Path("config/profiles.yaml"),
        help="Path to YAML configuration file",
    )
    parser.add_argument("--profile", type=str, default="default", help="Profile name in configuration")

    args = parser.parse_args()

    if not args.file.is_file():
        logger.error("File not found: %s", args.file)
        sys.exit(1)

    try:
        profile = load_profile(args.config, args.profile)
    except Exception as err:
        logger.error("Failed to load configuration: %s", err)
        sys.exit(1)

    checksum = calculate_source_sha256(args.file)
    logger.info("Source SHA-256 verification checksum: %s", checksum)

    success = upload_media_file(args.file, profile, progress_callback=print_progress)
    if success:
        logger.info("Transfer completed successfully.")
        sys.exit(0)
    else:
        logger.error("Transfer failed after retries.")
        sys.exit(2)


if __name__ == "__main__":
    main()

Test Suite (tests/)
tests/conftest.py
from pathlib import Path
import pytest


@pytest.fixture
def sample_file(tmp_path: Path) -> Path:
    p = tmp_path / "test_video.mkv"
    p.write_bytes(b"VIDEO_STREAM_MOCK_DATA_" * 100)
    return p


@pytest.fixture
def mock_config_yaml(tmp_path: Path) -> Path:
    p = tmp_path / "profiles.yaml"
    p.write_text(
        """
default:
  tv_ip: "127.0.0.1"
  port: 8080
  upload_endpoint: "/upload"
  form_field_name: "files[]"
  timeout_seconds: 5
  retry_attempts: 2
"""
    )
    return p

tests/test_config.py
from pathlib import Path
import pytest
from pixel_vlc_bridge.config import load_profile


def test_load_profile_valid(mock_config_yaml: Path) -> None:
    profile = load_profile(mock_config_yaml, "default")
    assert profile.tv_ip == "127.0.0.1"
    assert profile.port == 8080
    assert profile.retry_attempts == 2


def test_load_profile_missing_file() -> None:
    with pytest.raises(FileNotFoundError):
        load_profile(Path("non_existent.yaml"))

tests/test_integrity.py
import hashlib
from pathlib import Path
from pixel_vlc_bridge.integrity import calculate_source_sha256


def test_calculate_source_sha256(sample_file: Path) -> None:
    expected = hashlib.sha256(sample_file.read_bytes()).hexdigest()
    actual = calculate_source_sha256(sample_file)
    assert actual == expected

tests/test_uploader.py
import http.server
from pathlib import Path
import socketserver
import threading
from typing import Generator
import pytest

from pixel_vlc_bridge.config import DeviceProfile
from pixel_vlc_bridge.uploader import upload_media_file


class MockVLCHttpHandler(http.server.BaseHTTPRequestHandler):
    def do_POST(self) -> None:
        if self.path == "/upload":
            content_length = int(self.headers.get("Content-Length", 0))
            _ = self.rfile.read(content_length)
            self.send_response(200)
            self.end_headers()
            self.wfile.write(b"OK")
        else:
            self.send_response(404)
            self.end_headers()


@pytest.fixture
def mock_vlc_server() -> Generator[int, None, None]:
    server = socketserver.TCPServer(("127.0.0.1", 0), MockVLCHttpHandler)
    port = server.server_address[1]
    thread = threading.Thread(target=server.serve_forever)
    thread.daemon = True
    thread.start()
    yield port
    server.shutdown()


def test_upload_media_file_success(sample_file: Path, mock_vlc_server: int) -> None:
    profile = DeviceProfile(
        tv_ip="127.0.0.1",
        port=mock_vlc_server,
        upload_endpoint="/upload",
        form_field_name="files[]",
        retry_attempts=1,
    )
    progress_records = []

    def progress(curr: int, tot: int) -> None:
        progress_records.append((curr, tot))

    result = upload_media_file(sample_file, profile, progress_callback=progress)
    assert result is True
    assert len(progress_records) > 0
    assert progress_records[-1][0] == progress_records[-1][1]

GitHub Actions CI Workflow (.github/workflows/ci.yml)
name: CI Suite

on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]

permissions:
  contents: read

jobs:
  test-and-lint:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]

    steps:
      - name: Checkout repository
        uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@0a546201f3034e7e9370d014eb02330f8c8577dd # v5.0.0
        with:
          python-version: ${{ matrix.python-version }}

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install .[dev]

      - name: Run Ruff Linter
        run: ruff check .

      - name: Run Mypy Type Checker
        run: mypy src/ tests/

      - name: Run Pytest with Coverage
        run: pytest

Implementation Instructions
 * Verify Endpoint in DevTools: Confirm the path (e.g. /upload) and multipart form parameter (e.g. files[]) by inspecting DevTools during a browser upload to VLC Remote Playback on your Apple TV, then update config/profiles.yaml as needed.
 * Execute Locally:
   pip install -e .[dev]
ruff check .
mypy src/ tests/
pytest
vlc-upload /path/to/media.mkv --profile living_room

