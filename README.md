I've written the full design the prompt asked for and built Phase 1, a camera-check script. The script has not run on a Raspberry Pi yet, because this environment has no Pi or camera. I tested it here with no camera present (it fails cleanly with install instructions) and against a fake Picamera2 module (every camera check passes). Run it on your Pi and send me the output before we move to Phase 2.

The full design is saved as `/agent/surveillance/docs/DESIGN.md` and the script as `/agent/surveillance/tools/phase1_camera_check.py`. The main points are below.

---

# Part A: Design, before any recording code

## 0. Compatibility decisions

| Item | Decision |
|---|---|
| OS | **Raspberry Pi OS 64-bit Lite**. The current release is Debian 13 "Trixie"; Debian 12 "Bookworm" also works. Bullseye and older are not supported. |
| Python | System Python 3.11+ (3.11 on Bookworm, 3.13 on Trixie). |
| Camera API | **Picamera2** on **libcamera**, installed with apt (`python3-picamera2`), never with pip. |
| Deprecated | `picamera` v1, `raspistill`, `raspivid`, MMAL and `start_x=1` are the removed legacy stack. The replacements are **Picamera2** (Python) and **`rpicam-*`** (command line; `libcamera-*` are old aliases). `vcgencmd get_camera` only checks the legacy stack, so don't use it as a test. |
| OpenCV | `python3-opencv` from apt, which matches the system numpy that Picamera2 uses. |
| FFmpeg | From apt, used only to cut the video into files. It never re-encodes. |
| H.264 | The Pi 4 has a **hardware H.264 encoder** (up to 1080p30) and a hardware JPEG encoder. The Pi 5 has no hardware H.264 encoder. |
| Services | systemd with a watchdog, automatic restart and sandboxing. |
| Filesystem | ext4 only, never exFAT/FAT for recordings. |

**Why Picamera2:**
- **Only one process can use the CSI camera at a time.** Picamera2 lets that one process feed recording, motion detection and live view together.
- **One capture gives two streams.** The camera's image processor scales each frame in hardware into a full-quality **main** stream (for recording) and a small **lores** stream (for motion detection). That is exactly the requested split, with no second decode.
- It connects directly to the hardware encoders.
- `rpicam-vid` would hold the camera, leaving nothing for motion detection. OpenCV's `VideoCapture` can't use libcamera cameras properly.

## 1. Architecture

There are two systemd services, so a web bug or crash can never stop recording.

```text
surveillance-recorder.service   (owns camera, NO network access, writes recordings + DB)
  camera.py → recorder.py → ffmpeg segmenter (stream copy)
           ├→ motion_detector.py → database.py (SQLite)
           ├→ streaming.py (live frames, only while someone watches)
  storage_manager.py · control.py (Unix socket) · main.py (supervisor + watchdog)
                 ▲ Unix socket + read-only DB + recordings mounted READ-ONLY
surveillance-web.service        (no camera, binds 127.0.0.1:8080 only)
  web.py (Flask + Waitress) · authentication.py · security.py
                 ▲
  tailscale serve  →  HTTPS with a valid certificate, reachable only on your tailnet
                 ▲ WireGuard
  Your phone/laptop (Tailscale app + browser + app login)
```

- **Code** goes in `/opt/surveillance`, owned by root, so the service cannot modify it.
- **Data** goes in `/var/lib/surveillance/{config,database,recordings}`.
- **`recordings_dir`** is a single setting. Database paths are stored relative to it, so moving to an SSD later means copying the folder and changing that one setting.
- **Future features have hooks built in:**
  - a `camera_id` column in every table;
  - an event bus (motion started/ended, segment completed, storage low, camera offline) that notifications, backup and AI modules can subscribe to;
  - a detector interface for adding object detection later;
  - a `role` column on users for multiple users later.

## 2. Technology choices

| Concern | Choice |
|---|---|
| Recording | Hardware H.264 with a keyframe every 2 s, fed to one long-running `ffmpeg -f segment -c copy`. There is no re-encoding and no gap between files. ffmpeg reports each *completed* segment, so the file still being written is always known. |
| File format | **Fragmented MP4**, which stays playable up to about the last 2 s after a power cut. A normal MP4 is unplayable if it was never finished. |
| Names | `recordings/2026-09-23/00-05-00Z.mp4` in **UTC**, because local time repeats an hour when daylight saving ends. The browser shows local time. |
| Motion | OpenCV on the grey (Y) channel of the lores stream, scaled down to 320×180, at about 5 fps. |
| Database | SQLite in WAL mode with `synchronous=FULL`. It writes very little: one row per segment and one per event. |
| Web | Flask + Waitress, server-rendered pages, no CDN, a strict Content Security Policy. Status updates use `fetch` only while the tab is visible. |
| Passwords | Argon2id (`python3-argon2`). |
| Remote access | **Tailscale** + `tailscale serve`, so no router ports are opened. |

**Live view choice: on-demand MJPEG from the lores stream (640×360), sent through the logged-in web app.**
- Latency is about 0.2–0.5 s, and it works in every browser.
- It uses the Pi's hardware JPEG encoder, which **runs only while someone is watching**.
- It reuses the same login session, with no extra server program.
- **Rejected for version 1:**
  - HLS: 4–10 s delay.
  - WebRTC via MediaMTX: lowest delay at full resolution, but it needs an extra program, separate authentication and extra network ports. It is the recommended later upgrade.
  - RTSP: browsers can't play it.
- The cost is about 1–3 Mbit/s while viewing. Recording quality is never affected.

## 3. Data flow

```text
Camera → libcamera image processor (hardware, one capture)
   ├─ main 1920×1080 → H.264 HW encoder → ffmpeg segmenter → recordings/*.mp4 ─┐
   └─ lores 640×360 ─┬→ motion detector (320×180, 5 fps) → motion events ────┤→ SQLite
                     └→ JPEG HW encoder (only while viewers > 0) → web → browser
storage manager ←→ SQLite (quota, oldest-first deletion)   web ← SQLite (read-only)
```

Playback: the browser asks for `/recordings/<id>/video`. The app looks the id up in the database and serves the file with Range support, so the video player can stream and seek without a download. Paths never come from the browser.

## 4. Security model

**Layers (defence in depth):**
- **Nothing is exposed to the internet.** No port forwarding, no UPnP, no public IP.
- **Tailnet access rules.** Only your devices can reach the Pi, and only on port 443 (web) and 22 (SSH). The Pi cannot open connections to your other devices.
- **Firewall** blocks all incoming traffic except on the Tailscale interface.
- **The app listens on 127.0.0.1 only.** `tailscale serve` provides the HTTPS certificate in front of it. **Never enable Tailscale Funnel**, which publishes the service to the internet.
- **Unprivileged service user.** Both services run as `survcam` with no sudo. The web service sees recordings read-only, so even if it is compromised it cannot delete footage. The recorder has no network access.
- **No upload feature, no shell commands.** Reboot and shutdown use a fixed, polkit-authorised call.

**An app login is still needed even with the VPN.** Tailscale authenticates *devices*, not *people*. Without the login, any of these would give full access:
- a lost or unlocked phone;
- a family member's laptop on the same tailnet;
- malware on any tailnet device;
- a leaked Tailscale key;
- a mistake in the access rules.

The login also blocks attacks from malicious websites you visit while connected (CSRF, DNS rebinding).

**Authentication:**
- **No crypto-style private key as the password.** A long random string pasted into a box is just a password that's harder to use. The real version of the "private key" idea is **passkeys (WebAuthn)**, which I recommend as a later upgrade.
- **The admin account is created over SSH from the command line.** There is no web "first-run setup" page that a stranger could reach first.
- **Passwords** must be at least 15 characters and are checked against a common-password list. They are hashed with Argon2id (64 MiB memory, t=3, p=4), with a cap on how many hashes run at once. Unknown usernames still run a dummy hash, so the response doesn't reveal which usernames exist.
- **Brute-force protection:** failures are stored in the database, so restarts don't reset them. Each failure increases the delay, up to a 15-minute lockout, and there is a global cap on attempts per minute.
- **Sessions:**
  - The cookie is `__Host-sid`, set `Secure; HttpOnly; SameSite=Strict`.
  - The database stores only a hash of the session token.
  - Sessions expire after 30 minutes idle or 12 hours total.
  - A new session ID is issued at login, and there is a "log out everywhere" option.
- **CSRF tokens** plus an Origin check, a Host-header allow-list and strict security headers.
- **Re-enter your password** to reboot, shut down, change the password, or lower the storage limit (which deletes footage).
- **An audit log** records logins, failures, settings changes, downloads and reboots.

**SSH:**
- Reachable **only over Tailscale**, using ed25519 keys only.
- `PasswordAuthentication no`, `KbdInteractiveAuthentication no`, `PermitRootLogin no`, `AllowUsers <admin>`.
- The admin user is separate from the service user, which has no login shell.
- You administer remotely with `ssh admin@cam01` from any device on your tailnet.

## 5. Threat model (summary)

| Threat | Mitigated by | Not fully prevented |
|---|---|---|
| Unauthorised remote access / exposed ports | No public ports, tailnet rules, localhost-only app, firewall, app login | Someone who has a tailnet device *and* your password |
| Leaked VPN credentials / stolen phone | App login; revoke the device in Tailscale; optional Tailnet Lock | A browser that is already logged in, until its session expires |
| Weak password / brute force | 15+ characters, blocklist, Argon2id, backoff and lockout | Reusing a password leaked from another site; attackers locking *you* out |
| Session hijacking / CSRF / XSS | Secure cookies, rotation, timeouts, CSRF tokens, CSP, auto-escaping | Malware on your own device |
| Command injection / path traversal / unauthorised downloads | No shell, validated settings, files addressed by database id with a containment check, every route needs a login | Mistakes in future code |
| Malicious uploads | No upload feature exists | — |
| Stolen Pi or SD card | Only password hashes stored; revoke the Pi in Tailscale; change the Wi-Fi password | **The thief can watch the recordings.** Full-disk encryption conflicts with unattended reboot, because the Pi has no TPM. The thief also takes the footage of themselves, so the real fix is an off-device backup of motion clips. |
| Power cut, jamming, covered lens | Recording is local and doesn't need the network | Recording stops without power; a UPS helps |

The system is **not "100% secure"**. The biggest remaining risks are your own devices being compromised, physical theft, and future configuration mistakes.

## 6. Storage model

**Recording and the index:**
- Each segment is added to the database only after ffmpeg finishes it. The file currently being written is **never deleted**.
- Every 30 s, while usage is above `max_storage` **or** free space is below `min_free_space`, the storage manager deletes the oldest finished segment (file first, then its database row).
- **Retention setting:** `oldest_first` (default) or `prefer_motion`, which deletes non-motion footage first up to a set share.
- **If free space stays below half the emergency threshold,** recording pauses rather than filling the disk, and the dashboard shows a warning.
- **The recorder refuses to write into an unmounted SSD mount point.** Otherwise it would silently fill the SD card.

**Surviving power loss:**
- **ext4** journalling keeps the filesystem consistent.
- **Fragmented MP4** keeps the interrupted segment playable.
- **At startup,** any unfinished files are checked with `ffprobe` and marked recovered or corrupt.
- **SQLite:** WAL mode and a quick integrity check at every start. A nightly backup copy is kept. If the database is corrupt, the backup is restored and the recordings folder is rescanned. The files are the source of truth, so one bad database never loses footage.
- **Settings** are saved atomically (write a temporary file, `fsync`, then rename).
- **Clock:** the Pi 4 has **no real-time clock**. After a power cut without internet, timestamps are wrong until the clock syncs, so those segments are flagged.

The schema has these tables: `cameras`; `recordings` (with times, duration, size, status, a motion flag and a clock-synced flag); `motion_events`; and `motion_event_recordings`. That last one is a link table, because one event can span two segments. The full SQL is in `DESIGN.md`. Login and session data is kept separately in `auth.db`.

**Storage per bitrate:** GB per hour = Mbit/s × 0.45, so **45 GB lasts 100 ÷ Mbit/s hours**.

| Profile | Bitrate | GB/day | 45 GB lasts |
|---|---|---|---|
| 720p @ 10 fps | 1 Mbit/s | 10.8 | ~4.2 days |
| 720p @ 15 fps | 1.5 Mbit/s | 16.2 | ~2.8 days |
| 720p @ 30 fps | 3 Mbit/s | 32.4 | ~1.4 days |
| 1080p @ 10 fps | 2 Mbit/s | 21.6 | ~2.1 days |
| **1080p @ 15 fps (default)** | **3 Mbit/s** | **32.4** | **~33 h** |
| 1080p @ 30 fps | 5 Mbit/s | 54 | ~20 h |
| 1080p @ 30 fps, high quality | 8 Mbit/s | 86.4 | ~12.5 h |

- **One week** at the default profile needs about 230 GB; **30 days** needs about 1 TB.
- **SD card wear:** at the default profile, a 45 GB allocation on a 64 GB card is overwritten about every 1.4 days, roughly **12 TB of writes per year**. Ordinary cards can die within months, and "High/Max Endurance" cards last much longer. A **USB 3 SSD** (a typical 250 GB model is rated around 150 TB of writes) is strongly preferred for recordings.

## 7. Estimated resource use (to be measured in Phase 15)

| Component | CPU (share of one core) | RAM |
|---|---|---|
| Hardware H.264 + scaling | 3–8 % | camera buffers |
| ffmpeg segmenter | 2–5 % | 20–30 MB |
| Motion detection (320×180, 5 fps) | 5–10 % | ~10 MB |
| Recorder process | — | 80–150 MB |
| Web service (idle) | ~1 % | 40–70 MB |
| Live view, one viewer | +5–10 % while viewing | +10 MB |
| Tailscale | 1–3 % | 30–50 MB |
| **Total** | **~15–30 % of one core** (about 5–10 % of the whole CPU) | **~250–350 MB** |

Running motion detection on full 1080p30 frames would take about a whole core; the lores stream at 5 fps is about 100× less work. A 2 GB Pi 4 is enough. Use a heatsink or fan, because 24/7 use in a case can reach the 80 °C throttling point.

Motion detection works like this:
1. Blur the small grey frame, then compare it to a slowly updating background.
2. Mark pixels that differ by more than the sensitivity threshold, and measure the changed area.
3. Motion must appear in 3 frames in a row before an event starts.
4. The event stays open while movement continues and ends after `cooldown` seconds without motion, so one continuous movement produces one event.
5. If more than about 60 % of the frame changes at once, it is treated as a lighting change (lights switched on, IR filter, exposure jump) and the background is reset instead.

**Reliability:**
- If frames stop arriving or ffmpeg dies, the recorder stops sending its watchdog signal and systemd restarts it.
- If the camera disappears, it retries with increasing delays.
- Network loss has no effect on recording, and Wi-Fi power-saving is disabled.

## 8. Potential problems

1. No real-time clock, so timestamps are wrong after a power cut without internet.
2. SD card wear, and corruption from low voltage (the most common Pi failure).
3. Wi-Fi dropouts or jamming. Recording continues, but live view doesn't.
4. Seeking in fragmented MP4 depends on the browser. Phase 4 tests this on your devices, with a fallback ready.
5. Timestamps may drift if frames are dropped between the encoder and ffmpeg. Phase 3–4 checks this.
6. Running the H.264 and JPEG hardware encoders at the same time is unverified. Phase 10 checks it, with a software fallback.
7. False motion from trees, shadows, rain, insects near IR lights, headlights and exposure changes. It needs tuning, and later, masks.
8. Night vision requires a NoIR camera and an IR light.
9. 45 GB is only about 1.4 days at 1080p15.
10. The Tailscale certificate hostname appears in public certificate logs. Use a neutral name like `cam01`, not your address.
11. The recorder process is a single point of failure. Automatic restarts and the watchdog limit the impact.
12. OS upgrades can change camera behaviour. Test after major upgrades.
13. CCTV and privacy law: in the UK/EU (GDPR), recording public or neighbouring areas creates obligations.

**Recommended improvements, most valuable first:**
1. A USB 3 SSD for recordings.
2. Wired Ethernet if possible.
3. An encrypted off-device backup of motion clips.
4. An alert if the camera stops working, such as an external heartbeat check.
5. A real-time clock module and a UPS.
6. Passkeys or TOTP, plus Tailnet Lock.
7. Motion masks, and a NoIR camera with an IR light.
8. WebRTC live view later.

---

# Part B: Phase 1, verify camera access

## What we're building

A read-only diagnostic script. It confirms the operating system, Python, the modern camera stack, camera detection, the dual-stream mode (1080p main plus 640×360 lores), real frame delivery, the hardware encoders, power and temperature. It writes nothing to disk.

## Step 1: Prepare the SD card (in Raspberry Pi Imager on your computer)

1. Choose **Raspberry Pi 4** and **Raspberry Pi OS Lite (64-bit)**.
2. Under *Edit settings*, set:
   - **Hostname:** `cam01` (a neutral name).
   - **Username and password:** your own admin user, not `pi`.
   - **Wi-Fi:** your network name, password and country.
   - **Timezone.**
3. Under *Services*, choose **Enable SSH → Allow public-key authentication only**. Paste your public key; if you don't have one, create it on your computer with:

```bash
ssh-keygen -t ed25519 -C "laptop-to-cam01"
cat ~/.ssh/id_ed25519.pub
```

4. **With the Pi powered off**, connect the camera ribbon cable to the connector labelled CAMERA. Push it fully in and close the latch. Check the orientation diagram for your camera model.
5. Insert the card, power on with the official 5.1 V 3 A supply, and wait about 2 minutes.

## Step 2: Log in and update (SSH over your home network, for now only)

```bash
ssh youruser@cam01.local
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

## Step 3: Install the camera packages

```bash
sudo apt install -y python3-picamera2 --no-install-recommends
command -v rpicam-hello || sudo apt install -y rpicam-apps-lite
sudo apt install -y v4l-utils
```

(`--no-install-recommends` avoids pulling in desktop GUI packages on Lite.)

## Step 4: Quick check with the official tool

```bash
rpicam-hello --list-cameras
rpicam-hello -n -t 3000
```

The first command should list your camera. For example, a Camera Module 3 shows:

```text
Available cameras
-----------------
0 : imx708 [4608x2592 10-bit RGGB] (/base/soc/i2c0mux/i2c@1/imx708@1a)
```

The second command runs the camera for 3 seconds with no preview window and should exit without errors.

## Step 5: Create the Phase 1 script

The file goes at `~/surveillance/tools/phase1_camera_check.py`. Create it, then paste the code below:

```bash
mkdir -p ~/surveillance/tools
nano ~/surveillance/tools/phase1_camera_check.py
```

In `nano`, paste, then press Ctrl+O, Enter, Ctrl+X.

```python
#!/usr/bin/env python3
"""Phase 1: verify that the Raspberry Pi camera stack is usable from Python.

Run on the Raspberry Pi as your normal (non-root) user:

    python3 ~/surveillance/tools/phase1_camera_check.py

Exit code 0 means the camera is usable (warnings may still be printed).
Exit code 1 means at least one required check failed.
Nothing is written to disk.
"""
from __future__ import annotations

import argparse
import glob
import grp
import importlib.metadata
import os
import platform
import shutil
import subprocess
import sys
import time
from pathlib import Path

# Must be set before libcamera is loaded, otherwise it prints many INFO lines.
os.environ.setdefault("LIBCAMERA_LOG_LEVELS", "*:WARN")

MIN_PYTHON = (3, 11)
SUPPORTED_CODENAMES = {"bookworm", "trixie"}
KNOWN_SENSOR_OVERLAYS = ("ov5647", "imx219", "imx296", "imx477", "imx500", "imx708")

THROTTLE_BITS = {
    0: "under-voltage NOW",
    1: "ARM frequency capped NOW",
    2: "throttled NOW",
    3: "soft temperature limit NOW",
    16: "under-voltage has occurred since boot",
    17: "ARM frequency capping has occurred since boot",
    18: "throttling has occurred since boot",
    19: "soft temperature limit has occurred since boot",
}


class Report:
    def __init__(self) -> None:
        self.counts = {"PASS": 0, "WARN": 0, "FAIL": 0, "INFO": 0}

    def _emit(self, status: str, name: str, detail: str = "", hint: str = "") -> None:
        self.counts[status] += 1
        line = f"[{status}] {name}"
        if detail:
            line += f": {detail}"
        print(line, flush=True)
        if hint:
            for hint_line in hint.splitlines():
                print(f"       -> {hint_line}", flush=True)

    def ok(self, name: str, detail: str = "") -> None:
        self._emit("PASS", name, detail)

    def warn(self, name: str, detail: str = "", hint: str = "") -> None:
        self._emit("WARN", name, detail, hint)

    def fail(self, name: str, detail: str = "", hint: str = "") -> None:
        self._emit("FAIL", name, detail, hint)

    def info(self, name: str, detail: str = "") -> None:
        self._emit("INFO", name, detail)

    @property
    def failed(self) -> bool:
        return self.counts["FAIL"] > 0


def section(title: str) -> None:
    print(f"\n=== {title} ===", flush=True)


def read_text(path: str) -> str | None:
    try:
        return Path(path).read_text(errors="replace").replace("\x00", "").strip()
    except OSError:
        return None


def parse_size(value: str) -> tuple[int, int]:
    try:
        width, height = (int(part) for part in value.lower().split("x"))
    except ValueError:
        raise argparse.ArgumentTypeError(f"expected WIDTHxHEIGHT, got {value!r}") from None
    if not (16 <= width <= 4096 and 16 <= height <= 4096):
        raise argparse.ArgumentTypeError(f"size out of range: {value!r}")
    return width, height


def run_command(args: list[str], timeout: float = 10.0) -> subprocess.CompletedProcess[str] | None:
    if shutil.which(args[0]) is None:
        return None
    try:
        return subprocess.run(args, capture_output=True, text=True, timeout=timeout, check=False)
    except (OSError, subprocess.TimeoutExpired):
        return None


def os_release() -> dict[str, str]:
    values: dict[str, str] = {}
    text = read_text("/etc/os-release") or ""
    for line in text.splitlines():
        key, sep, value = line.partition("=")
        if sep:
            values[key.strip()] = value.strip().strip('"')
    return values


def check_platform(report: Report) -> None:
    section("Platform")

    model = read_text("/proc/device-tree/model")
    if model is None:
        report.warn("Board", "not a Raspberry Pi (no /proc/device-tree/model)",
                    "This script is meant to run on the Raspberry Pi itself.")
    elif "Raspberry Pi 4" in model:
        report.ok("Board", model)
    elif "Raspberry Pi 5" in model:
        report.warn("Board", model,
                    "The Pi 5 has no hardware H.264 encoder; this project is tuned for the Pi 4.")
    else:
        report.warn("Board", model, "This project is designed and tested for the Raspberry Pi 4.")

    release = os_release()
    pretty = release.get("PRETTY_NAME", "unknown")
    codename = release.get("VERSION_CODENAME", "").lower()
    if codename in SUPPORTED_CODENAMES:
        report.ok("Operating system", pretty)
    elif codename == "bullseye":
        report.warn("Operating system", pretty,
                    "Bullseye is outdated. Reinstall with the current 64-bit Raspberry Pi OS.")
    else:
        report.warn("Operating system", pretty,
                    "Expected Raspberry Pi OS Bookworm or Trixie.")

    machine = platform.machine()
    if machine == "aarch64":
        report.ok("CPU architecture", "aarch64 (64-bit)")
    else:
        report.warn("CPU architecture", machine, "A 64-bit Raspberry Pi OS is recommended.")

    report.info("Kernel", platform.release())

    version = ".".join(str(part) for part in sys.version_info[:3])
    if sys.version_info >= MIN_PYTHON:
        report.ok("Python", f"{version} ({sys.executable})")
    else:
        report.warn("Python", version, "Python 3.11 or newer is expected.")

    if sys.prefix != sys.base_prefix:
        report.warn("Python environment", f"running inside a virtualenv: {sys.prefix}",
                    "For Phase 1 run with the system python3 (/usr/bin/python3).\n"
                    "Later phases use a venv created with --system-site-packages.")

    if os.geteuid() == 0:
        report.warn("User", "running as root",
                    "Run as your normal user. The service will never run as root.")
    else:
        try:
            video_gid = grp.getgrnam("video").gr_gid
        except KeyError:
            video_gid = None
        if video_gid is not None and video_gid not in os.getgroups():
            report.warn("User groups", "current user is not in the 'video' group",
                        "sudo usermod -aG video $USER   (then log out and back in)")
        else:
            report.ok("User groups", "current user can access video devices")


def check_boot_config(report: Report) -> None:
    section("Boot configuration")

    config_path = next((p for p in ("/boot/firmware/config.txt", "/boot/config.txt")
                        if Path(p).is_file()), None)
    if config_path is None:
        report.warn("config.txt", "not found in /boot/firmware or /boot")
        return

    active_lines = []
    for raw in (read_text(config_path) or "").splitlines():
        line = raw.split("#", 1)[0].strip().replace(" ", "")
        if line:
            active_lines.append(line)

    if "start_x=1" in active_lines:
        report.fail("Legacy camera stack", f"start_x=1 is set in {config_path}",
                    "Remove the start_x=1 line (legacy MMAL stack) and reboot.")
    else:
        report.ok("Legacy camera stack", "not enabled (good)")

    overlays = [line for line in active_lines
                if line.startswith("dtoverlay=") and any(s in line for s in KNOWN_SENSOR_OVERLAYS)]
    if "camera_auto_detect=1" in active_lines:
        report.ok("Camera auto-detect", f"camera_auto_detect=1 in {config_path}")
    elif overlays:
        report.ok("Camera overlay", ", ".join(overlays))
    else:
        report.warn("Camera auto-detect", f"camera_auto_detect=1 not found in {config_path}",
                    "Add camera_auto_detect=1 (or the dtoverlay for your sensor) and reboot.")


def check_power_and_temperature(report: Report) -> None:
    section("Power and temperature")

    raw_temp = read_text("/sys/class/thermal/thermal_zone0/temp")
    if raw_temp and raw_temp.isdigit():
        celsius = int(raw_temp) / 1000
        if celsius >= 80:
            report.warn("CPU temperature", f"{celsius:.1f} C",
                        "The Pi throttles at 80 C. Add a heatsink/fan or improve airflow.")
        elif celsius >= 70:
            report.warn("CPU temperature", f"{celsius:.1f} C", "Warm. Cooling is recommended for 24/7 use.")
        else:
            report.ok("CPU temperature", f"{celsius:.1f} C")
    else:
        report.warn("CPU temperature", "unavailable")

    result = run_command(["vcgencmd", "get_throttled"])
    if result is None or result.returncode != 0 or "=" not in result.stdout:
        report.warn("Throttling status", "vcgencmd unavailable")
        return
    try:
        value = int(result.stdout.strip().split("=", 1)[1], 16)
    except ValueError:
        report.warn("Throttling status", f"unexpected output: {result.stdout.strip()}")
        return
    if value == 0:
        report.ok("Throttling status", "throttled=0x0 (no under-voltage or throttling)")
        return
    problems = [text for bit, text in THROTTLE_BITS.items() if value & (1 << bit)]
    hint = ""
    if value & 0x10001:
        hint = ("Under-voltage causes SD-card corruption and camera errors.\n"
                "Use the official 5.1 V / 3 A USB-C power supply and a short cable.")
    report.warn("Throttling status", f"0x{value:x}: " + "; ".join(problems), hint)


def check_hardware_codecs(report: Report) -> None:
    section("Hardware video encoders")

    names = {read_text(path) for path in glob.glob("/sys/class/video4linux/video*/name")}
    names.discard(None)
    if "bcm2835-codec-encode" in names:
        report.ok("H.264 hardware encoder", "bcm2835-codec-encode present")
    else:
        report.warn("H.264 hardware encoder", "bcm2835-codec-encode not found",
                    "Expected on a Pi 4 running Raspberry Pi OS. Recording would need software encoding.")
    if "bcm2835-codec-encode_image" in names:
        report.ok("JPEG hardware encoder", "bcm2835-codec-encode_image present")
    else:
        report.info("JPEG hardware encoder", "not found (live view would use software JPEG)")


def check_tools(report: Report) -> None:
    section("Supporting tools (needed in later phases)")

    rpicam = shutil.which("rpicam-hello") or shutil.which("libcamera-hello")
    if rpicam:
        report.ok("rpicam-apps", rpicam)
    else:
        report.warn("rpicam-apps", "rpicam-hello not found",
                    "sudo apt install -y rpicam-apps-lite   (useful for troubleshooting)")

    ffmpeg = shutil.which("ffmpeg")
    if ffmpeg:
        result = run_command(["ffmpeg", "-hide_banner", "-version"])
        first_line = result.stdout.splitlines()[0] if result and result.stdout else ffmpeg
        report.ok("FFmpeg", first_line)
    else:
        report.info("FFmpeg", "not installed yet (installed in Phase 3)")

    try:
        import cv2  # noqa: PLC0415
        report.ok("OpenCV", cv2.__version__)
    except ImportError:
        report.info("OpenCV", "not installed yet (installed in Phase 6)")


def package_version(name: str) -> str:
    try:
        return importlib.metadata.version(name)
    except importlib.metadata.PackageNotFoundError:
        return "unknown version"


def check_camera(report: Report, args: argparse.Namespace) -> None:
    section("Camera (libcamera + Picamera2)")

    try:
        from picamera2 import Picamera2  # noqa: PLC0415
    except ImportError as exc:
        report.fail("Picamera2 import", str(exc),
                    "sudo apt install -y python3-picamera2 --no-install-recommends\n"
                    "Do NOT install picamera2 with pip; the apt package matches the system libcamera.")
        return
    report.ok("Picamera2 import", package_version("picamera2"))

    cameras = Picamera2.global_camera_info()
    if not cameras:
        report.fail("Camera detection", "libcamera found no cameras",
                    "1. Power off, reseat the ribbon cable at both ends (contacts the right way round).\n"
                    "2. Run: rpicam-hello --list-cameras\n"
                    "3. Run: dmesg | grep -iE 'imx|ov5647|unicam|csi'\n"
                    "Note: 'vcgencmd get_camera' reports the LEGACY stack and is not a valid test.")
        return
    for info in cameras:
        report.ok(f"Camera {info.get('Num', cameras.index(info))}",
                  f"model={info.get('Model')} location={info.get('Location')} id={info.get('Id')}")
    if args.camera >= len(cameras):
        report.fail("Camera selection", f"--camera {args.camera} requested but only {len(cameras)} found")
        return

    try:
        picam2 = Picamera2(args.camera)
    except (RuntimeError, IndexError) as exc:
        report.fail("Open camera", str(exc),
                    "The camera is probably in use by another program.\n"
                    "Check with: pgrep -af 'rpicam|libcamera|picamera2|motion'")
        return

    try:
        exercise_camera(report, picam2, args)
    finally:
        for step in (picam2.stop, picam2.close):
            try:
                step()
            except Exception:  # noqa: BLE001 - best-effort cleanup
                pass


def exercise_camera(report: Report, picam2, args: argparse.Namespace) -> None:
    props = picam2.camera_properties
    report.info("Sensor", f"{props.get('Model')} pixel array {props.get('PixelArraySize')}")

    if args.modes:
        # Slow: Picamera2 briefly configures every sensor mode to measure it.
        for mode in picam2.sensor_modes:
            report.info("Sensor mode",
                        f"{mode.get('size')} @ {mode.get('fps', 0):.1f} fps, "
                        f"{mode.get('bit_depth')}-bit {mode.get('format')}")

    config = picam2.create_video_configuration(
        main={"size": args.main, "format": "YUV420"},
        lores={"size": args.lores, "format": "YUV420"},
        controls={"FrameRate": args.fps},
    )
    picam2.align_configuration(config)
    main_size = tuple(config["main"]["size"])
    lores_size = tuple(config["lores"]["size"])
    if main_size != args.main or lores_size != args.lores:
        report.info("Size alignment", f"main {args.main}->{main_size}, lores {args.lores}->{lores_size}")

    try:
        picam2.configure(config)
    except Exception as exc:  # noqa: BLE001 - libcamera raises several exception types
        report.fail("Configure dual-stream video mode", str(exc),
                    "Try a smaller --main size or lower --fps.")
        return
    report.ok("Configure dual-stream video mode",
              f"main {main_size[0]}x{main_size[1]} YUV420 + lores {lores_size[0]}x{lores_size[1]} YUV420")

    current = picam2.camera_configuration() if hasattr(picam2, "camera_configuration") else config
    sensor = current.get("sensor") or {}
    if sensor:
        report.info("Selected sensor mode",
                    f"output_size={sensor.get('output_size')} bit_depth={sensor.get('bit_depth')}")

    try:
        picam2.start()
    except Exception as exc:  # noqa: BLE001
        report.fail("Start camera", str(exc))
        return
    report.ok("Start camera")

    time.sleep(1.0)  # let auto-exposure and white balance settle

    timestamps = []
    metadata = {}
    deadline = time.monotonic() + max(10.0, args.frames / args.fps * 3)
    while len(timestamps) < args.frames and time.monotonic() < deadline:
        metadata = picam2.capture_metadata()
        if "SensorTimestamp" in metadata:
            timestamps.append(metadata["SensorTimestamp"])

    if len(timestamps) < 2:
        report.fail("Frame delivery", f"only {len(timestamps)} frames received",
                    "The camera started but frames are not arriving. Check dmesg for CSI errors.")
        return

    elapsed_s = (timestamps[-1] - timestamps[0]) / 1e9
    measured_fps = (len(timestamps) - 1) / elapsed_s if elapsed_s > 0 else 0.0
    detail = f"{len(timestamps)} frames, measured {measured_fps:.1f} fps (requested {args.fps:g})"
    if measured_fps >= args.fps * 0.8:
        report.ok("Frame delivery", detail)
    else:
        report.warn("Frame delivery", detail,
                    "Low light makes auto-exposure use longer frames, which lowers the frame rate.\n"
                    "Repeat the test in normal room lighting.")

    report.info("Exposure",
                f"exposure={metadata.get('ExposureTime')} us "
                f"analogue_gain={_fmt(metadata.get('AnalogueGain'))} "
                f"lux={_fmt(metadata.get('Lux'))} "
                f"colour_temp={metadata.get('ColourTemperature')} K")

    check_lores_brightness(report, picam2)


def _fmt(value) -> str:
    return f"{value:.2f}" if isinstance(value, (int, float)) else str(value)


def check_lores_brightness(report: Report, picam2) -> None:
    buffer = picam2.capture_buffer("lores")
    stream = picam2.stream_configuration("lores")
    width, height = stream["size"]
    stride = stream["stride"]
    # YUV420: the first stride*height bytes are the Y (brightness) plane.
    luma = buffer[: stride * height].reshape(height, stride)[:, :width]
    mean = float(luma.mean())
    detail = f"{width}x{height} Y-plane mean brightness {mean:.0f}/255"
    if mean < 8:
        report.warn("Low-resolution frame", detail,
                    "Image is almost black: lens cap on, camera facing a wall, or a dark room.")
    elif mean > 247:
        report.warn("Low-resolution frame", detail, "Image is almost white: overexposed.")
    else:
        report.ok("Low-resolution frame", detail)


def build_arg_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(description="Phase 1: verify Raspberry Pi camera access.")
    parser.add_argument("--camera", type=int, default=0, help="camera number (default 0)")
    parser.add_argument("--main", type=parse_size, default=(1920, 1080),
                        help="recording stream size (default 1920x1080)")
    parser.add_argument("--lores", type=parse_size, default=(640, 360),
                        help="motion/live-view stream size (default 640x360)")
    parser.add_argument("--fps", type=float, default=15.0, help="frame rate (default 15)")
    parser.add_argument("--frames", type=int, default=45, help="frames to measure (default 45)")
    parser.add_argument("--modes", action="store_true", help="also list all sensor modes (slow)")
    return parser


def main() -> int:
    args = build_arg_parser().parse_args()
    if not 1 <= args.fps <= 60:
        print("--fps must be between 1 and 60", file=sys.stderr)
        return 2
    if args.lores[0] > args.main[0] or args.lores[1] > args.main[1]:
        print("--lores must not be larger than --main", file=sys.stderr)
        return 2

    report = Report()
    try:
        check_platform(report)
        check_boot_config(report)
        check_power_and_temperature(report)
        check_hardware_codecs(report)
        check_tools(report)
        check_camera(report, args)
    except KeyboardInterrupt:
        print("\nInterrupted.", file=sys.stderr)
        return 130

    counts = report.counts
    section("Summary")
    print(f"PASS={counts['PASS']} WARN={counts['WARN']} FAIL={counts['FAIL']} INFO={counts['INFO']}")
    if report.failed:
        print("RESULT: FAILED - fix the [FAIL] items above, then run this script again.")
        return 1
    print("RESULT: CAMERA READY - Phase 1 complete.")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

## Step 6: Run the test

Use the system `python3`, not a virtualenv, for this phase.

```bash
python3 ~/surveillance/tools/phase1_camera_check.py
python3 ~/surveillance/tools/phase1_camera_check.py --modes     # also lists sensor modes
```

## Expected output (roughly)

```text
=== Platform ===
[PASS] Board: Raspberry Pi 4 Model B Rev 1.5
[PASS] Operating system: Debian GNU/Linux 13 (trixie)
[PASS] CPU architecture: aarch64 (64-bit)
[PASS] Python: 3.13.x (/usr/bin/python3)
...
=== Hardware video encoders ===
[PASS] H.264 hardware encoder: bcm2835-codec-encode present
[PASS] JPEG hardware encoder: bcm2835-codec-encode_image present
=== Camera (libcamera + Picamera2) ===
[PASS] Camera 0: model=imx708 ...
[PASS] Configure dual-stream video mode: main 1920x1080 YUV420 + lores 640x360 YUV420
[PASS] Start camera
[PASS] Frame delivery: 45 frames, measured 15.0 fps (requested 15)
[PASS] Low-resolution frame: 640x360 Y-plane mean brightness 110/255
=== Summary ===
RESULT: CAMERA READY - Phase 1 complete.
```

`[INFO] FFmpeg/OpenCV not installed yet` is normal at this stage.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `libcamera found no cameras` | Power off, reseat the ribbon at both ends, check `dmesg \| grep -iE 'imx\|ov5647\|unicam'`. |
| `Open camera` fails (busy) | Another program has the camera. Run `pgrep -af 'rpicam\|libcamera\|picamera2'` and stop it. |
| `start_x=1` FAIL | Remove that line from `/boot/firmware/config.txt`, then reboot. |
| Under-voltage WARN | Use the official power supply. This is the number one cause of SD card corruption. |
| Low fps WARN | Normal in dim light, because auto-exposure lengthens each frame. Retest in room light. Night settings are handled in later phases. |
| `No module named picamera2` | `sudo apt install -y python3-picamera2 --no-install-recommends`. Don't use pip. |

**Send me the full output of Step 4 and Step 6**, including which camera module you have and your OS version. I won't start Phase 2 (still-image capture) until Phase 1 passes on your Pi.
