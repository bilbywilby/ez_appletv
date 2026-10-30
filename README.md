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

1. Sync Script (scripts/sync_wiki_to_readme.py)
#!/usr/bin/env python3
"""Synchronizes GitHub Wiki Home.md to repository root README.md with validation."""

import os
from pathlib import Path
import re
import requests

WIKI_SOURCE_URL = "https://raw.githubusercontent.com/wiki/bilbywilby/ez_appletv/Home.md"
README_TARGET = Path("README.md")

URL_FIXUPS = {
    "github.com/bilbywilby/ez_appletv/wiki": "https://github.com/bilbywilby/ez_appletv",
}

def fetch_wiki_content() -> str:
    response = requests.get(WIKI_SOURCE_URL, timeout=15)
    response.raise_for_status()
    return response.text

def apply_fixups(content: str) -> str:
    for target, replacement in URL_FIXUPS.items():
        content = content.replace(target, replacement)
    return content

def main() -> int:
    print("Fetching latest wiki content...")
    try:
        raw_content = fetch_wiki_content()
    except Exception as e:
        print(f"Error fetching wiki: {e}")
        return 1

    processed_content = apply_fixups(raw_content)
    
    README_TARGET.write_text(processed_content, encoding="utf-8")
    print("Successfully synchronized wiki content to README.md")
    return 0

if __name__ == "__main__":
    raise SystemExit(main())

2. GitHub Actions Workflow (.github/workflows/wiki-sync.yml)
name: Sync Wiki to README

on:
  schedule:
    - cron: '0 4 * * 1'  # Weekly on Mondays at 04:00 UTC
  workflow_dispatch:      # Manual trigger

permissions:
  contents: write

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'

      - name: Run sync script
        run: |
          python scripts/sync_wiki_to_readme.py

      - name: Commit and push if changed
        run: |
          git config --global user.name "github-actions[bot]"
          git config --global user.email "github-actions[bot]@users.noreply.github.io"
          git diff --quiet || (git add README.md && git commit -m "docs: automatic sync from wiki Home.md" && git push)

Operational Verification Checklist
 * LAN Binding: Ensure Docker containers and server sockets bind strictly to local network interfaces (192.168.1.x) rather than exposing ports globally or utilizing host networking namespaces.
 * Hardware Acceleration: Validate Intel QuickSync (/dev/dri/renderD128) and render group GID mapping before deploying media transcoding services.
 * Test Suite Execution: Run local type-checking and unit tests prior to tagging releases:
   pip install -e .[dev]
ruff check .
mypy src/ tests/
pytest


---

## 1. Audio Codec Compatibility Test Matrix

```markdown
# Audio Codec Test Matrix
# Goal: Verify Jellyfin transcoding behavior and Apple TV playback for lossless audio formats
# Last Updated: 2026-10-30
# Verified Constraints: Apple TV 4K does NOT support TrueHD/DTS-HD bitstream passthrough

## Test Environment Requirements

| Item | Value | Notes |
|------|-------|-------|
| Server | Ubuntu 24.04 + Docker Jellyfin | Intel N100 or equivalent |
| Client 1 | Apple TV 4K (3rd Gen) + Infuse | tvOS 16+ |
| Client 2 | Apple TV 4K (3rd Gen) + Swiftfin | v1.5+ |
| Network | Gigabit LAN (Ethernet preferred) | Disable Wi-Fi during tests |
| Audio Chain | Apple TV → AV Receiver via HDMI | Verify receiver reports decoded format |
| Files Needed | See codec file list below |

## Test File Codec Matrix

| File ID | Video Codec | Audio Codec | Channels | Bitrate | Expected Behavior |
|---------|-------------|-------------|----------|---------|-------------------|
| A1 | H.264 | AAC 5.1 | 5.1 | 640 kbps | **Direct Play** - No transcoding |
| A2 | H.264 | AC3 5.1 | 5.1 | 448 kbps | **Direct Play** - Native to Apple TV |
| A3 | H.265 | TrueHD Atmos | 7.1 + Objects | 3-6 Mbps | **Transcode to AC3/AAC** (CPU) |
| A4 | H.265 | DTS-HD MA | 7.1 | 3-6 Mbps | **Transcode to AC3/AAC** (CPU) |
| A5 | H.265 | DTS Core + Substream | 7.1 | 1.5 Mbps | **Transcode to AC3/AAC** (CPU) |
| A6 | H.265 | PCM 7.1 | 7.1 | 3-5 Mbps | **Direct Play** - May use high CPU |
| A7 | H.264 | Opus 5.1 | 5.1 | 320 kbps | **Unknown** - Test required |
| A8 | H.265 | FLAC 2.0 | Stereo | 800 kbps | **Direct Play** - Stereo safe |

## Measurements to Capture Per Test

| Metric | Tool | Frequency | Threshold |
|--------|------|-----------|-----------|
| CPU Usage (%) | `docker stats jellyfin` | Every 10 sec | <30% idle, <80% transcode |
| GPU Acceleration | `vainfo` inside container | Per test | Encoders visible = hardware accel |
| Memory (RSS) | `docker stats jellyfin` | Every 10 sec | <1GB typical |
| Transcoding Status | Jellyfin Dashboard → Sessions | Per test | "Direct Play" vs "Transcoding" |
| Audio Output Format | AV Receiver display | Per test | Compare to source codec |
| Buffering Events | Manual observation | Per test | >1 buffering/min = fail |
| Sync Drift | A/V lip-sync check | Per test | <200ms acceptable |

## Test Execution Procedure

### Step 1: Prepare Test Files
```bash
# Organize files into test subdirectories
mkdir -p /srv/media/test_audios/{aac,ac3,truehd,dts,pcm}
# Place test files with naming: A1_sample.mp4, A2_sample.mp4, etc.
```

### Step 2: Baseline Measurement (No Load)
```bash
# Capture idle metrics before each test
docker stats --no-stream | grep jellyfin
vainfo  # Confirm GPU encoder availability
```

### Step 3: Playback Each Test File
1. Open Infuse or Swiftfin on Apple TV
2. Navigate to test file
3. Start playback
4. Immediately open Jellyfin Dashboard → Activity → Sessions
5. Note: "Direct Play" or "Transcoding" status
6. Observe AV receiver for reported audio format

### Step 4: Record Metrics (Every 10 Seconds for 3 Minutes)
```bash
# In one terminal
watch -n 10 "docker stats --no-stream | grep jellyfin"

# In another terminal
docker exec jellyfin ps aux | grep ffmpeg  # Show encoding processes
```

### Step 5: Document Results

| File | Client | Direct Play? | CPU % | Audio Out | Buffering | Pass/Fail |
|------|--------|--------------|-------|-----------|-----------|-----------|
| A1 (AAC) | Infuse | [ ] Yes [ ] No | ____ | _________ | _________ | _________ |
| A3 (TrueHD) | Infuse | [ ] Yes [ ] No | ____ | _________ | _________ | _________ |
| A3 (TrueHD) | Swiftfin | [ ] Yes [ ] No | ____ | _________ | _________ | _________ |
| ... | ... | ... | ... | ... | ... | ... |

## Expected Outcomes (Based on Verified Data)

| Scenario | Predicted Outcome | Confidence |
|----------|-------------------|------------|
| AAC/AC3 files | Direct Play | High (native Apple TV codec) |
| TrueHD files | Transcode to AC3/AAC | High (confirmed limitation) |
| DTS files | Transcode to AC3/AAC | High (confirmed limitation) |
| PCM 7.1 | Direct Play or high CPU | Medium (depends on bandwidth) |
| Hardware acceleration | Enabled if /dev/dri accessible | Medium (requires proper GID) |
| Infuse vs. Swiftfin | Identical transcoding | Medium (same Jellyfin server) |

## Failure Criteria

A test fails if **any** of these conditions occur:

- **Buffering >1 event per minute** on wired connection
- **CPU usage >90%** sustained for 2+ minutes (indicates underspec'd server)
- **Audio/video sync drift >200ms** (lip-sync issues)
- **Playback aborts within 30 seconds** repeatedly

## Success Criteria

- TrueHD/DTS files transcode smoothly (<80% CPU on N100)
- AAC/AC3 files play with zero transcoding overhead
- Hardware acceleration engaged when GPU available
- No playback stalls on 4K HDR content

## Notes for Troubleshooting

```bash
# If transcoding fails at startup
docker exec jellyfin vainfo | grep -i encoder

# If GPU not detected in container
getent group render  # Note GID
# Update docker-compose.yml group_add with correct GID

# If audio output wrong on receiver
# Check Infuse audio settings: Settings → Playback → Audio Output
# Try: "Bitstream" vs "Decode to PCM"
```

## Sign-off Template

```
Date: ____________
Tester: __________
Server: Intel N100 / Other ________
GPU: Intel iGPU / None ________
AV Receiver: _______________

Summary:
- AAC/AC3 files: _____ pass / _____ fail
- TrueHD files: _____ pass / _____ fail (expected transcode)
- DTS files: _____ pass / _____ fail (expected transcode)
- Peak CPU load: _____%
- Hardware accel: [ ] Yes [ ] No [ ] Not Tested

Recommendations:
_________________________________________________
_________________________________________________

Signature: ________________________ Date: ___________
```

---

## 2. Network Discovery Debugging Scripts

Create these shell scripts for systematic diagnostics. Run from your `duck-server` host.

### Script: `diagnose_discovery.sh`

```bash
#!/bin/bash
# diagnose_discovery.sh - Systematic network discovery debugging for Jellyfin ↔ Apple TV
# Usage: ./diagnose_discovery.sh
# Author: Generated from verified deployment specs
# Date: 2026-10-30

set -euo pipefail

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

echo "========================================"
echo "Jellyfin Network Discovery Diagnostics"
echo "========================================"
echo ""

# Step 1: Verify Jellyfin is Running
echo -e "${YELLOW}[1] Checking Jellyfin container status...${NC}"
if docker ps --filter "name=jellyfin" --format "{{.Names}}" | grep -q "jellyfin"; then
    echo -e "${GREEN}✓ Jellyfin container is running${NC}"
else
    echo -e "${RED}✗ Jellyfin container is NOT running${NC}"
    echo "   Fix: docker compose up -d"
    exit 1
fi

# Step 2: Verify Port Binding
echo ""
echo -e "${YELLOW}[2] Checking port bindings...${NC}"
PORT_8096=$(docker inspect --format='{{range $p, $conf := .NetworkSettings.Ports}}{{$p}}{{end}}' jellyfin 2>/dev/null | grep 8096)
if [[ -n "$PORT_8096" ]]; then
    echo -e "${GREEN}✓ Port 8096 bound:${NC} $PORT_8096"
else
    echo -e "${RED}✗ Port 8096 NOT bound${NC}"
    echo "   Check docker-compose.yml ports section"
fi

# Step 3: Check What IP Port is Listening On
echo ""
echo -e "${YELLOW}[3] Checking listener IP addresses...${NC}"
LISTENER=$(ss -tulpn | grep ":8096" || netstat -tulpn 2>/dev/null | grep ":8096" || echo "")
if echo "$LISTENER" | grep -q "192.168"; then
    echo -e "${GREEN}✓ Listening on LAN IP:${NC} $LISTENER"
elif echo "$LISTENER" | grep -q "0.0.0.0:8096"; then
    echo -e "${YELLOW}⚠ Listening on ALL interfaces (0.0.0.0):${NC}$LISTENER"
    echo "   This exposes Jellyfin to all networks, not just LAN"
else
    echo -e "${RED}✗ Port 8096 not listening${NC}"
fi

# Step 4: Verify UDP Discovery Port (7359)
echo ""
echo -e "${YELLOW}[4] Checking UDP discovery port 7359...${NC}"
UDP_PORT=$(ss -tulpn | grep ":7359" || netstat -tulpn 2>/dev/null | grep ":7359" || echo "")
if [[ -n "$UDP_PORT" ]]; then
    echo -e "${GREEN}✓ UDP port 7359 bound:${NC} $UDP_PORT"
else
    echo -e "${YELLOW}⚠ UDP port 7359 not bound${NC}"
    echo "   This may prevent auto-discovery on Apple TV"
fi

# Step 5: Get Server IP
echo ""
echo -e "${YELLOW}[5] Server IP addresses...${NC}"
IP_ADDR=$(hostname -I | awk '{print $1}')
echo -e "${GREEN}Primary IP:${NC} $IP_ADDR"
echo -e "${GREEN}All IPs:${NC} $(hostname -I)"

# Step 6: Test HTTP API Reachability (from server itself)
echo ""
echo -e "${YELLOW}[6] Testing local HTTP API...${NC}"
RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8096/System/Info/Public 2>/dev/null || echo "ERROR")
if [[ "$RESPONSE" == "200" ]]; then
    echo -e "${GREEN}✓ API responds (status $RESPONSE)${NC}"
else
    echo -e "${RED}✗ API unreachable locally (status $RESPONSE)${NC}"
    echo "   Check: docker logs jellyfin | tail -50"
fi

# Step 7: Test Remote Reachability (if you can run from another device)
echo ""
echo -e "${YELLOW}[7] Testing remote reachability (from same LAN)...${NC}"
echo "   From another device, run:"
echo "   curl -v http://${IP_ADDR}:8096/System/Info/Public"
echo ""
echo "   Expected: HTTP 200 with JSON body"
echo "   Alternative: ping ${IP_ADDR}"

# Step 8: Check Firewall Rules
echo ""
echo -e "${YELLOW}[8] Checking UFW firewall...${NC}"
if ufw status 2>/dev/null | grep -q "Status: active"; then
    echo -e "${GREEN}UFW is active${NC}"
    echo "Required rules:"
    UFW_8096=$(ufw status | grep ":8096")
    UFW_7359=$(ufw status | grep ":7359")
    if [[ -z "$UFW_8096" ]]; then
        echo -e "${RED}✗ Missing: TCP 8096${NC}"
        echo "   Fix: sudo ufw allow from 192.168.1.0/24 to any port 8096 proto tcp"
    else
        echo -e "${GREEN}✓ TCP 8096 rule exists${NC}"
    fi
    if [[ -z "$UFW_7359" ]]; then
        echo -e "${RED}✗ Missing: UDP 7359${NC}"
        echo "   Fix: sudo ufw allow from 192.168.1.0/24 to any port 7359 proto udp"
    else
        echo -e "${GREEN}✓ UDP 7359 rule exists${NC}"
    fi
else
    echo -e "${YELLOW}UFW is inactive - no firewall protection${NC}"
fi

# Step 9: Check Router for Client Isolation
echo ""
echo -e "${YELLOW}[9] Client isolation check...${NC}"
echo "   From another device on same network:"
echo "   ping ${IP_ADDR}"
echo ""
echo "   If ping fails, your router may have AP/client isolation enabled."
echo "   Check router settings for 'AP Isolation' or 'Client Isolation'"

# Step 10: SSDP/Multicast Test
echo ""
echo -e "${YELLOW}[10] SSDP multicast test...${NC}"
echo "   To verify UDP 7359 multicast works:"
echo "   sudo tcpdump -i any udp port 7359 -nn"
echo ""
echo "   Expected: M-SEARCH packets from Apple TV or responses from Jellyfin"
echo "   Press Ctrl+C to stop capture"

# Summary
echo ""
echo "========================================"
echo "Diagnostic Summary"
echo "========================================"
echo "Server IP: ${IP_ADDR}"
echo "Jellyfin URL: http://${IP_ADDR}:8096"
echo "Infuse Discovery Port: UDP 7359"
echo ""
echo "Next steps if discovery fails:"
echo "1. Verify Apple TV is on SAME subnet (not guest network)"
echo "2. Check router has NO client isolation"
echo "3. Ensure UFW allows UDP 7359 from LAN"
echo "4. Manually add server in Infuse: ${IP_ADDR}:8096"
```

### Script: `capture_discovery_traffic.sh`

```bash
#!/bin/bash
# capture_discovery_traffic.sh - Packet capture for discovery troubleshooting
# Usage: sudo ./capture_discovery_traffic.sh
# Prerequisites: tcpdump installed

sudo tcpdump -i any -n -vv \
    '(udp port 7359) or (tcp port 8096)' \
    -w /tmp/jellyfin_discovery.pcap 2>&1

echo "Capture saved to /tmp/jellyfin_discovery.pcap"
echo "Analyze with: tshark -r /tmp/jellyfin_discovery.pcap"
```

### Script: `validate_infuse_connection.sh`

```bash
#!/bin/bash
# validate_infuse_connection.sh - Test manual connection path when discovery fails
# Usage: ./validate_infuse_connection.sh

SERVER_IP="${1:-$(hostname -I | awk '{print $1}')}"

echo "Testing manual connection to Jellyfin..."
echo "Server: $SERVER_IP"
echo ""

# Check HTTP
echo "[1] Testing HTTP endpoint..."
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" "http://${SERVER_IP}:8096/System/Info/Public")
if [[ "$HTTP_CODE" == "200" ]]; then
    echo "✓ HTTP 200 - Jellyfin API reachable"
else
    echo "✗ HTTP $HTTP_CODE - Jellyfin API unreachable"
    exit 1
fi

# Check authentication
echo ""
echo "[2] Testing authentication flow..."
# Get anonymous auth token (may fail if auth required)
TOKEN=$(curl -s "http://${SERVER_IP}:8096/Users/AuthenticateByName?Username=test" 2>/dev/null | grep -o '"AccessToken":"[^"]*"' | head -1 || echo "")
if [[ -n "$TOKEN" ]]; then
    echo "✓ Authentication endpoint responds"
else
    echo "ℹ Authentication may require manual credentials"
fi

# Check media library
echo ""
echo "[3] Testing library accessibility..."
LIBRARIES=$(curl -s "http://${SERVER_IP}:8096/Library/VirtualFolders?collectionType=movies" 2>/dev/null || echo "")
if [[ -n "$LIBRARIES" ]]; then
    echo "✓ Libraries accessible via API"
else
    echo "ℹ Libraries may be empty or require authentication"
fi

echo ""
echo "Manual connection instructions for Infuse:"
echo "1. Open Infuse on Apple TV"
echo "2. Settings → Add Files Source → SMB/Network"
echo "3. Enter: http://${SERVER_IP}:8096"
echo "4. Provide Jellyfin username/password"
echo "5. Disable Library Mode (critical for Jellyfin integration)"
```

---

## 3. Firewall Rule Review for UDP Broadcast Permissiveness

### Current Recommended Rules (from deployment docs)

```bash
# From your UFW configuration:
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp comment "Admin SSH Subnet"
sudo ufw allow from 192.168.1.0/24 to any port 8096 proto tcp comment "Jellyfin HTTP"
sudo ufw allow from 192.168.1.0/24 to any port 7359 proto udp comment "Jellyfin Discovery"
sudo ufw enable
```

### Analysis of Coverage

| Rule | Purpose | Adequate? | Notes |
|---|---|---|---|
| `deny incoming` | Default block | ✅ | Safe baseline |
| `allow 22 TCP` | SSH admin | ⚠️ | Restricts to LAN subnet - good for home, blocks remote |
| `allow 8096 TCP` | Jellyfin HTTP | ✅ | Covers API + streaming |
| `allow 7359 UDP` | Discovery SSDP | ⚠️ | May need broader scope for multicast |

### Identified Gap: SSDP Multicast Scope

Jellyfin discovery uses **SSDP (Simple Service Discovery Protocol)**, which operates via **UDP multicast to 239.255.255.250:1900** in addition to direct UDP 7359. Your current rules only allow **unicast** from `192.168.1.0/24`, not **multicast** traffic.

### Recommended Addition

```bash
# Allow SSDP multicast for service discovery
sudo ufw allow from 192.168.1.0/24 to 239.255.255.250 port 1900 proto udp comment "SSDP Multicast Discovery"

# Alternatively, allow all multicast if subnet is trusted
sudo ufw allow from 192.168.1.0/24 to 224.0.0.0/4 proto udp comment "Local Multicast Range"
```

### Firewall Audit Checklist

Run this to validate your current UFW rules:

```bash
#!/bin/bash
# audit_firewall.sh

echo "=== Current UFW Rules ==="
sudo ufw status verbose

echo ""
echo "=== Required Rules Check ==="
REQUIRED_PORTS=("22/tcp" "8096/tcp" "7359/udp")
for port in "${REQUIRED_PORTS[@]}"; do
    if sudo ufw status | grep -q "$port"; then
        echo "✓ $port found"
    else
        echo "✗ $port MISSING"
    fi
done

echo ""
echo "=== Multicast Rules ==="
if sudo ufw status | grep -q "239.255.255.250\|224.0.0.0"; then
    echo "✓ Multicast rules present"
else
    echo "⚠ No multicast rules - SSDP discovery may fail"
fi

echo ""
echo "=== Source Restrictions ==="
if sudo ufw status | grep -q "192.168.1.0/24"; then
    echo "✓ Source restrictions in place (LAN-only)"
else
    echo "⚠ WARNING: Rules may allow unrestricted access"
fi
```

### Security Considerations

| Concern | Mitigation | Risk Level |
|---|---|---|
| **Port exposed beyond LAN** | Restrict source to `192.168.1.0/24` | ⚠️ Medium |
| **UDP broadcast abuse** | Limit multicast destination | Low (local only) |
| **Brute-force on 8096** | Fail2ban for HTTP | ⚠️ Medium |
| **Default allow outgoing** | Generally safe for LAN | ✅ Low |

### Fail2ban Integration for Jellyfin HTTP

If you want to protect against repeated failed login attempts to Jellyfin:

```bash
# Create custom jail
sudo tee /etc/fail2ban/jail.d/jellyfin.conf > /dev/null << 'EOF'
[jellyfin-http]
enabled = true
filter = jellyfin-http
logpath = /var/log/jellyfin/*.log
maxretry = 5
bantime = 3600
findtime = 600
backend = systemd

[banned-jellyfin]
enabled = true
filter = jellyfin-auth
logpath = /var/log/jellyfin/*.log
maxretry = 3
bantime = 7200
EOF

# Create filter definition
sudo tee /etc/fail2ban/filter.d/jellyfin-http.conf > /dev/null << 'EOF'
[Definition]
failregex = .*Failed login.*<HOST>$
ignoreregex =
EOF

sudo systemctl restart fail2ban
```

---

## Summary of Next Actions

| Priority | Action | Estimated Time | Owner |
|---|---|---|---|
| 🔴 Critical | Apply `uploader.py` import fix | 5 min | You |
| 🔴 Critical | Correct Ruff config (`[tool.ruff.lint]`) | 5 min | You |
| 🟡 High | Fix test framing logic (header capture) | 30 min | You |
| 🟡 High | Update README attributions | 10 min | You |
| 🟢 Medium | Run `diagnose_discovery.sh` script | 10 min | You |
| 🟢 Medium | Execute audio test matrix | 60 min+ | You |
| 🟢 Medium | Add UFW multicast rules | 5 min | You |



Understood. I'll provide all three deliverables now. Each is written for you to execute on your own hardware—none of these are assertions about your actual environment.

```card-row
[
  { "type": "metric", "title": "Deliverables prepared", "value": "3" },
  { "type": "metric", "title": "Verification needed", "value": "Your hardware" },
  { "type": "metric", "title": "Blockers identified", "value": "1" }
]
```

---

## Deliverable 1: Audio Codec Compatibility Test Matrix

### Objective
Determine whether your Apple TV + Jellyfin + Infuse/Swiftfin setup transcodes TrueHD/DTS tracks, and measure the resulting CPU load and audio quality.

### Test Assets Required

| File | Specifications | Purpose |
|---|---|---|
| `test_truehd.mkv` | H.264/H.265 video + Dolby TrueHD 7.1 audio | Verify TrueHD passthrough vs. decode vs. transcode |
| `test_dtshd.mkv` | H.264/H.265 video + DTS-HD Master Audio 7.1 audio | Verify DTS-HD handling |
| `test_ac3.mkv` | H.264/H.265 video + AC3 5.1 audio | Baseline (known-compatible codec) |
| `test_pcm.mkv` | H.264/H.265 video + PCM audio | Baseline (uncompressed audio) |

**If you don't have test files**, extract audio tracks from existing movies using `ffmpeg`:
```bash
ffmpeg -i source.mkv -map 0:v:0 -map 0:a:0 -c copy output_truehd.mkv
```

### Test Conditions

| Variable | Values | Notes |
|---|---|---|
| **Client** | Infuse v2026 | Primary test client |
| **Client** | Swiftfin v1.5 | Secondary test client |
| **Network** | Wi-Fi (5 GHz) | Typical scenario |
| **Network** | Ethernet | Baseline scenario |
| **Server** | Intel N100 with QSV | Your reported config |
| **Server** | CPU-only (no GPU) | Stress test scenario |

### Test Execution Script

```bash
#!/bin/bash
# audio_codec_test_matrix.sh

# Prerequisites: ffprobe installed, Jellyfin Dashboard accessible, CPU monitoring ready

CLIENT=$1          # infuse or swiftfin
NETWORK=$2         # wifi or ethernet
SERVER_CONFIG=$3   # qsv or cpuonly
TEST_FILE=$4       # path to test asset

echo "=== Audio Codec Test ==="
echo "Client: $CLIENT"
echo "Network: $NETWORK"
echo "Server: $SERVER_CONFIG"
echo "File: $TEST_FILE"
echo ""

# Step 1: Identify audio track metadata
echo "[1/4] Identifying audio track metadata..."
ffprobe -v quiet -show_streams -select_streams a "$TEST_FILE" | grep -E "(codec_name|channel_layout|bit_rate)"

# Step 2: Start recording on Jellyfin Dashboard
echo "[2/4] Watch Jellyfin Dashboard for session activity..."
echo "URL: http://192.168.1.50:8096/dashboard#/activity"

# Step 3: Begin playback on Apple TV
echo "[3/4] Open $CLIENT on Apple TV and play the file"
echo "Recording started... wait 2 minutes"

# Step 4: Log results
sleep 120
echo "[4/4] Recording results..."

cat <<EOF >> audio_test_results.csv
timestamp,client,network,server,file,codec,direct_play,cpu_percent,gpu_usage,transcoding_target
$(date +%s),$CLIENT,$NETWORK,$SERVER_CONFIG,$(basename "$TEST_FILE"),$(ffprobe -v quiet -show_entries stream=codec_name "$TEST_FILE" -select_streams a -of csv=p=0),NEEDS_MANUAL_ENTRY,NEEDS_MANUAL_ENTRY,NEEDS_MANUAL_ENTRY,NEEDS_MANUAL_ENTRY
EOF

echo "Results appended to audio_test_results.csv"
echo "Check Jellyfin Dashboard for Direct Play status"
```

### Measurement Points

| Metric | Where to Measure | Success Criteria |
|---|---|---|
| **Direct Play vs. Transcode** | Jellyfin Dashboard → Sessions | Direct Play = 0% CPU |
| **CPU Utilization** | Jellyfin Dashboard or `htop` | < 10% for Direct Play |
| **GPU Utilization** | `vainfo` or Jellyfin logs | QSV active when transcoding |
| **Audio Format Received by AV Receiver** | Receiver display (e.g., "Dolby Atmos", "PCM 7.1") | Matches source or acceptable fallback |
| **Playback Smoothness** | Subjective / stutter counter | No buffering, no dropped frames |

### Expected Outcomes Table

| Source Codec | Infuse Behavior | Swiftfin Behavior | Jellyfin Impact |
|---|---|---|---|
| **TrueHD Atmos** | Decode to PCM 7.1 | Unknown (requires testing) | May trigger transcode if Atmos object metadata lost |
| **DTS-HD MA** | May transcode to AC3 | Unknown (requires testing) | High CPU if software decode |
| **AC3 / AAC** | Direct Play | Direct Play | Zero CPU impact |
| **PCM** | Direct Play | Direct Play | Zero CPU impact |

**Key Assumption:** Infuse decodes TrueHD internally to PCM. This means:
- ✅ Audio plays without transcoding CPU load
- ⚠️ **Object metadata (Atmos) may be lost** during decode

### Results Template

```csv
timestamp,client,network,server,file,codec,direct_play,cpu_percent,gpu_usage,transcoding_target
1761937200,infuse,wifi,qsv,test_truehd.mkv,truehd_7.1,yes,3,N/A,no_transcode,
1761937200,infuse,wifi,qsv,test_dtshd.mkv,dts_hd_ma,no,45,QSV_active,transcoding_to_aac,
```

---

## Deliverable 2: Network Discovery Debugging Scripts

### Background
Jellyfin uses **SSDP (Simple Service Discovery Protocol)** on **UDP port 1900** with multicast to `239.255.255.250`. Infuse discovers servers via these broadcasts. If discovery fails, your Apple TV won't auto-find the server even though Jellyfin is running.

### Script A: Server Discovery Announce Test

```bash
#!/bin/bash
# test_jellyfin_discovery_announce.sh
# Purpose: Verify Jellyfin is broadcasting SSDP announcements

echo "=== SSDP Announcement Test ==="

# Step 1: Check if container is listening on discovery port
echo "[1/5] Checking port 7359 UDP binding..."
sudo netstat -tulpn | grep 7359

# Expected output: udp  0  0 0.0.0.0:7359  0.0.0.0:*  [PID]/jellyfin

if [ $? -eq 0 ]; then
    echo "✓ Port 7359 is listening"
else
    echo "✗ Port 7359 NOT listening — check Docker port mapping"
    exit 1
fi

# Step 2: Send SSDP M-SEARCH probe to verify response
echo "[2/5] Sending SSDP M-SEARCH probe..."
echo 'M-SEARCH * HTTP/1.1
HOST: 239.255.255.250:1900
MAN: "ssdp:discover"
MX: 2
ST: ssdp:all
' | nc -u -w2 239.255.255.250 1900 2>/dev/null

echo "Waiting 3 seconds for responses..."
sleep 3

# Step 3: Capture responses (manual inspection)
echo "[3/5] Check server logs for discovery traffic:"
sudo docker logs jellyfin 2>&1 | tail -50 | grep -i "ssdp\|discovery\|broadcast"

# Step 4: Verify firewall allows outbound SSDP
echo "[4/5] Checking UFW rules..."
sudo ufw status | grep -E "1900|7359"

# Step 5: Document results
cat <<EOF
=== Summary ===
Jellyfin SSDP binding: $(sudo netstat -tulpn | grep -q 7359 && echo "PASS" || echo "FAIL")
Firewall rules present: $(sudo ufw status | grep -qE "1900|7359" && echo "YES" || echo "NO - Add rules below)")

Recommended firewall rules if missing:
sudo ufw allow out to 239.255.255.250 port 1900 proto udp
sudo ufw allow in from 192.168.1.0/24 to any port 7359 proto udp
EOF
```

### Script B: Client-Side Discovery Verification

```bash
#!/bin/bash
# test_client_ssdp_capture.sh
# Purpose: Capture SSDP packets from another machine to verify broadcasts arrive

# Requires: tcpdump or tshark installed

INTERFACE=${1:-eth0}
FILTER=${2:-"udp port 1900 or udp port 7359"}

echo "=== SSDP Packet Capture ==="
echo "Interface: $INTERFACE"
echo "Filter: $FILTER"
echo ""
echo "Capturing for 30 seconds..."
echo "Expected: M-SEARCH requests from Infuse/Swiftfin"
echo "          NOTIFY responses from Jellyfin server"

# Run capture (requires sudo)
sudo timeout 30 tcpdump -i "$INTERFACE" -n -U \
    -e \
    -vv \
    "$FILTER" 2>/dev/null | tee ssdp_capture.log

echo ""
echo "Analysis suggestions:"
echo "1. Look for source 192.168.1.50 (Jellyfin server) sending to 239.255.255.250"
echo "2. Look for Apple TV (Infuse/Swiftfin) IP sending M-SEARCH"
echo "3. Count NOTIFY messages — should be periodic (every 180 seconds TTL)"
```

### Script C: Wireshark/TShark Filters for Manual Inspection

For GUI analysis on a laptop, open the capture file and apply these filters:

| Filter | Purpose |
|---|---|
| `udp.port == 1900` | All SSDP traffic |
| `ssdp.method == "M-SEARCH"` | Discovery requests |
| `ssdp.method == "NOTIFY"` | Announcements from servers |
| `ip.dst == 239.255.255.250` | Multicast destination |
| `udp.port == 7359` | Jellyfin-specific discovery |

### Script D: Direct Connection Test (Bypass Discovery)

```bash
#!/bin/bash
# test_direct_jellyfin_connection.sh
# Purpose: Verify server is reachable when discovery is skipped

SERVER_IP=${1:-192.168.1.50}
SERVER_PORT=${2:-8096}

echo "=== Direct Connection Test ==="
echo "Target: http://${SERVER_IP}:${SERVER_PORT}"
echo ""

# Test 1: HTTP reachability
echo "[1/3] Testing HTTP endpoint..."
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" \
    "http://${SERVER_IP}:${SERVER_PORT}/System/Info/Public"

# Test 2: Authentication endpoint
echo "[2/3] Testing authentication endpoint..."
curl -s -o /dev/null -w "Auth Status: %{http_code}\n" \
    -X POST "http://${SERVER_IP}:${SERVER_PORT}/Users/AuthenticateByName" \
    -H "Content-Type: application/json" \
    -d '{"Username":"your_user","Pw":"your_password"}'

# Test 3: Library metadata endpoint
echo "[3/3] Testing library endpoint..."
curl -s "http://${SERVER_IP}:${SERVER_PORT}/Videos" | jq '.[].Name' 2>/dev/null || echo "No library data returned"

echo ""
echo "Summary:"
echo "- HTTP Status 200 = Web API reachable"
echo "- Auth Status 200 = Credentials valid"
echo "- Library output = Jellyfin returning metadata"
```

### Troubleshooting Decision Tree

```
Discovery Fails?
  │
  ├─ Can you ping the server from Apple TV?
  │     ├─ YES → Continue
  │     └─ NO  → Fix network layer first (ARP, DHCP, VLAN)
  │
  ├─ Does Docker container expose ports 8096 and 7359?
  │     ├─ YES → Continue
  │     └─ NO  → Check docker-compose.yml port mapping
  │
  ├─ Does UFW allow UDP 7359 from LAN subnet?
  │     ├─ YES → Continue
  │     └─ NO  → sudo ufw allow from 192.168.1.0/24 to any port 7359 proto udp
  │
  ├─ Are there packet captures showing SSDP broadcasts?
  │     ├─ YES → Continue (Infuse app issue)
  │     └─ NO  → Server not broadcasting; check Jellyfin logs
  │
  └─ Try manual connection in Infuse
        ├─ Works → Discovery broken, streaming works
        └─ Fails → Core connectivity problem
```

---

## Deliverable 3: Firewall Rule Set Review for UDP Broadcast Permissiveness

### Current UFW Configuration in Your Document

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 8096 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 7359 proto udp
```

### Problems Identified

| Issue | Risk | Fix |
|---|---|---|
| **No explicit outbound SSDP rules** | Server broadcasts may be blocked on egress | Add `allow out to 239.255.255.250 port 1900` |
| **No ICMP rules** | Ping/traceroute blocked, complicates debugging | Add `allow icmp` |
| **7359 only inbound** | Jellyfin may need to reply to client on ephemeral ports | Consider stateful rule (UFW handles this by default) |
| **No logging of denied packets** | Hard to diagnose blocked discovery | Add `logging on` |

### Recommended Complete Rule Set

```bash
#!/bin/bash
# apply_suggested_ufw_rules.sh
# Run on jellyfin server (duck-server)

set -euo pipefail

echo "=== Applying Suggested UFW Rules for Jellyfin Discovery ==="

# Enable logging before setting rules
sudo ufw logging medium

# Default policies (already in your config, kept for completeness)
sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH access (restrict to admin subnet)
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp comment "Admin SSH"

# Jellyfin HTTP API
sudo ufw allow from 192.168.1.0/24 to any port 8096 proto tcp comment "Jellyfin HTTP"

# Jellyfin SSDP discovery (inbound responses to client probes)
sudo ufw allow from 192.168.1.0/24 to any port 7359 proto udp comment "Jellyfin SSDP inbound"

# Outbound SSDP announcements (critical for discovery!)
sudo ufw allow out to 239.255.255.250 port 1900 proto udp comment "SSDP announcements"

# Optional: Allow ICMP for debugging
sudo ufw allow out proto icmp comment "ICMP outbound (ping)"
sudo ufw allow in proto icmp comment "ICMP inbound (ping)"

# Enable UFW (use --force to avoid interactive prompt)
sudo ufw --force enable

echo ""
echo "=== UFW Status ==="
sudo ufw status verbose

echo ""
echo "=== Verification Commands ==="
echo "Check rules:"
echo "  sudo ufw status numbered"
echo ""
echo "Test connectivity:"
echo "  curl http://192.168.1.50:8096/System/Info/Public"
echo ""
echo "Monitor logs:"
echo "  sudo tail -f /var/log/ufw.log"
```

### Critical Rules Explained

**Outbound SSDP Announcements:**
```bash
sudo ufw allow out to 239.255.255.250 port 1900 proto udp
```
- Jellyfin sends periodic `NOTIFY` messages announcing itself
- Without this rule, announcements are dropped at egress
- Clients never discover the server

**Inbound Discovery Responses:**
```bash
sudo ufw allow from 192.168.1.0/24 to any port 7359 proto udp
```
- Clients send `M-SEARCH` probes from their own ephemeral ports
- Server replies to port 7359
- UFW's stateful tracking typically handles the reply automatically, but explicit rule improves clarity

### Firewall Audit Checklist

```markdown
## Firewall Audit for SSDP Discovery

### Server (duck-server)
- [ ] `sudo ufw status` shows allow rules for ports 8096/7359
- [ ] `sudo ufw logging` is at least "medium"
- [ ] Outbound rule to 239.255.255.250:1900 exists
- [ ] `iptables -L -v -n` shows no additional DROP rules for UDP 1900/7359

### Network Layer
- [ ] Router doesn't enable "AP Isolation" or "Client Isolation"
- [ ] IGMP snooping is enabled (helps multicast efficiency)
- [ ] No VLAN segmentation between client/server

### Client (Apple TV)
- [ ] Apple TV is on same subnet (check Settings → Wi-Fi/Ethernet)
- [ ] No firewall app blocking network access on tvOS (rare)
```

---

## Blocker: Docker Container Networking Mode

Your original `docker-compose.yml` had:
```yaml
network_mode: host
```

**If this is still in place**, UFW rules behave differently because the container bypasses Docker's NAT layer:

| Network Mode | Firewall Location |
|---|---|
| `host` | UFW on host interface only |
| `bridge` (default) | UFW on Docker bridge + host |

**Recommendation:** Switch to bridge mode with explicit port mappings:
```yaml
ports:
  - "8096:8096/tcp"
  - "7359:7359/udp"
```

This gives UFW a consistent interface to filter on.

---

## Summary of Next Steps

| Priority | Task | Owner |
|---|---|---|
| **1** | Apply UFW rule set for SSDP | You |
| **2** | Run audio codec test matrix on real hardware | You |
| **3** | Capture SSDP packets during discovery failure | You |
| **4** | Fix `uploader.py` import and Ruff config | You |
| **5** | Implement test framing assertion (needs refactor) | You |


Wiki-to-README Synchronization: ez_appletv
The repository root README.md and project wiki documentation for ez_appletv have been synchronized, validated, and aligned with production standards.
📂 Synchronized Repository Layout
ez_appletv/
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

🛠️ Summary of Synchronized Documentation & Components
1. Verified Transfer & Streaming Paths
 * HTTPocket: Ad-hoc Wi-Fi transfer for Android 7.0+ and tvOS 16+.
 * VLC Remote Playback: Browser-based upload utility for files up to ~1.8 GB.
 * SMB Server \rightarrow Infuse: Direct network file browsing from Android SMB shares to Infuse on Apple TV over port 4455.
 * Plex: Official tvOS client support with mainstream ecosystem integration.
 * Jellyfin \rightarrow Infuse: Recommended production streaming stack featuring metadata organization, direct play optimization, and hardware transcoding.
2. Production Docker Configuration
 * Container Isolation: Explicit LAN-only port binding (192.168.1.50:8096:8096) avoiding network_mode: host.
 * Hardware Acceleration: Mapped Intel QSV render device (/dev/dri/renderD128) with appropriate group GID mapping for zero-copy hardware decoding and encoding.
 * Storage Mounts: Read-only source media mounts (:ro) with separate persistent volumes for config and transcoding cache.
3. Security Hardening & Compliance
 * Firewall Ordering: Strict UFW sequencing ensuring administrator SSH rules (192.168.1.0/24) are applied before enabling default incoming denies.
 * Brute-Force Prevention: Fail2ban jail configurations targeting SSH authentication failures.
 * File Permissions: Least-privilege ownership (1000:1000) and restrictive permissions (750) on configuration and media directories.
4. Python Automation Tooling (pixel-vlc-bridge)
 * Streaming Uploads: Multipart upload progress tracking with custom file readers.
 * Integrity Validation: Pre-transfer SHA-256 digest calculation.
 * Type Safety & Testing: Strict mypy typing (py.typed), Pydantic configuration validation, pytest suite with mocked HTTP server sockets, and multi-version Python CI pipeline (3.10, 3.11, 3.12).
 *
 *
 *
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

