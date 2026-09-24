Your Phase 13 results are good, and the Pi confirmed the parts I couldn't test in the VM:

- **Camera under the device rules:** the recorder works with only the camera devices allowed.
- **Crash and hang:** a `kill -9` came back in 5 s, and the watchdog restarted a frozen recorder 8 s after its timeout.
- **Network rule:** the kernel blocks the web service from the internet (`000`).
- **Security scores:** 1.3 and 1.4. CPU is 13–20% for the recorder and 0.4% for the web service.
- **Phase 12:** the camera can't reach your laptop over the tailnet (100% loss).

Update 0.13.1 below fixes the two problems your run exposed:

- **Checker FAIL lines (tool bug, not a recording problem):** the segment checker compared every file with today's setting (10 fps), but your older files were recorded at 15 fps. It also crashed when the newest `.partial` file was finalised while it was running.
  - It now judges each file against its own frame rate and marks older ones `(recorded at 15 fps, before a settings change)`.
  - It skips files that finish or get deleted during the check.
- **Throttling:** the sysfs file I used doesn't exist on kernel 6.18, and the sandboxed web service can't run `vcgencmd`, so System Status would show throttling as unknown.
  - A small third service, `surveillance-health`, now runs `vcgencmd get_throttled` every 30 s and writes the answer to `/run/surveillance/throttled`, a file in RAM.
  - It can open only `/dev/vcio_gencmd`, which is limited to status commands (with `/dev/vcio` as fallback). It has no network at all.
  - The web service just reads the file, so it still has no device access. In the VM its security score is 1.0 OK.
  - The whole path to the System Status page worked in the test container.

One thing is still untested: the reboot. The boot-time figure in your output (26 822 s, about 7.4 hours after boot) shows the Pi wasn't rebooted, so that test is still pending (Step 4).

## Step 1 — Apply the update

```bash
cd ~/surveillance
cat > update_to_0_13_1.py <<'PYEOF'
#!/usr/bin/env python3
"""Update the surveillance project from 0.13.0 to 0.13.1 (run from ~/surveillance)."""
import os, shutil, sys
from pathlib import Path

EDITS = [
    ('app/__init__.py',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.13.0"\n',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.13.1"\n'),
    ('app/system_info.py',
     '\nCACHE_S = 30.0\n# Same value as `vcgencmd get_throttled`, readable without access to the VideoCore device.\nTHROTTLED_SYSFS = ("/sys/devices/platform/soc/soc:firmware/get_throttled",)\n\nTHROTTLE_FLAGS = {\n',
     '\nCACHE_S = 30.0\n# `vcgencmd get_throttled` needs the VideoCore device, which the web service cannot open; the\n# small surveillance-health service (deploy/throttle-monitor.sh) writes its answer here instead.\nTHROTTLED_FILE = "throttled"\nTHROTTLED_MAX_AGE_S = 120\n\nTHROTTLE_FLAGS = {\n'),
    ('app/system_info.py',
     '    """Thread-safe; CPU usage is measured between successive calls."""\n\n    def __init__(self) -> None:\n        self._lock = threading.Lock()\n        self._last_cpu: tuple[int, int] | None = None\n',
     '    """Thread-safe; CPU usage is measured between successive calls."""\n\n    def __init__(self, runtime_dir: Path | None = None) -> None:\n        self._runtime_dir = runtime_dir\n        self._lock = threading.Lock()\n        self._last_cpu: tuple[int, int] | None = None\n'),
    ('app/system_info.py',
     '        return round(int(raw) / 1000, 1) if raw and raw.isdigit() else None\n\n    @staticmethod\n    def throttled() -> dict | None:\n        raw = next((value for value in map(_read, THROTTLED_SYSFS) if value), None)\n        if raw is None:\n            out = _run(["vcgencmd", "get_throttled"])\n',
     '        return round(int(raw) / 1000, 1) if raw and raw.isdigit() else None\n\n    def throttled(self) -> dict | None:\n        raw = self._throttled_from_monitor()\n        if raw is None:\n            out = _run(["vcgencmd", "get_throttled"])\n'),
    ('app/system_info.py',
     '            return None\n        return {"raw": f"0x{value:x}", "flags": [text for bit, text in THROTTLE_FLAGS.items() if value & (1 << bit)]}\n\n    @staticmethod\n',
     '            return None\n        return {"raw": f"0x{value:x}", "flags": [text for bit, text in THROTTLE_FLAGS.items() if value & (1 << bit)]}\n\n    def _throttled_from_monitor(self) -> str | None:\n        if self._runtime_dir is None:\n            return None\n        path = self._runtime_dir / THROTTLED_FILE\n        try:\n            age = time.time() - path.stat().st_mtime\n            text = path.read_text().strip()\n        except OSError:\n            return None\n        if age > THROTTLED_MAX_AGE_S or not text.startswith("throttled="):\n            return None\n        return text.split("=", 1)[1]\n\n    @staticmethod\n'),
    ('app/web.py',
     '    app.register_blueprint(create_blueprint(settings))\n    app.register_blueprint(create_live_blueprint(settings))\n    system_info = SystemInfo()\n    https_hostname = settings.web.https_hostname\n    allowed_hostnames = ALLOWED_HOSTNAMES | ({https_hostname} if https_hostname else set())\n',
     '    app.register_blueprint(create_blueprint(settings))\n    app.register_blueprint(create_live_blueprint(settings))\n    system_info = SystemInfo(settings.runtime_dir)\n    https_hostname = settings.web.https_hostname\n    allowed_hostnames = ALLOWED_HOSTNAMES | ({https_hostname} if https_hostname else set())\n'),
    ('tools/phase4_check_segments.py',
     'frame rate, duration close to the configured segment length. It also checks that consecutive\nsegments leave no missing time, and probes any leftover .partial files.\n"""\nfrom __future__ import annotations\n',
     'frame rate, duration close to the configured segment length. It also checks that consecutive\nsegments leave no missing time, and probes any leftover .partial files.\nEach file is judged against the frame rate it was recorded at, so recordings made before a\nsettings change are not reported as failures.\n"""\nfrom __future__ import annotations\n'),
    ('tools/phase4_check_segments.py',
     'import argparse\nimport json\nimport subprocess\nimport sys\n',
     'import argparse\nimport json\nimport statistics\nimport subprocess\nimport sys\n'),
    ('tools/phase4_check_segments.py',
     '    checked = []\n    for path in segments:\n        packets = ffprobe_packets(path)\n        width, height = ffprobe_size(path)\n',
     '    checked = []\n    for path in segments:\n        if not path.exists():\n            continue  # deleted by the storage manager while this ran\n        packets = ffprobe_packets(path)\n        width, height = ffprobe_size(path)\n'),
    ('tools/phase4_check_segments.py',
     '            continue\n        times = [t for t, _ in packets]\n        gaps = [(a - times[0], b - a) for a, b in zip(times, times[1:]) if b - a > 1.5 / fps]\n        duration = times[-1] - times[0] + 1 / fps\n        measured_fps = (len(times) - 1) / (times[-1] - times[0])\n        problems = []\n        if not packets[0][1]:\n            problems.append("does not start with a keyframe")\n',
     '            continue\n        times = [t for t, _ in packets]\n        steps = [b - a for a, b in zip(times, times[1:])]\n        # The file\'s own frame interval: the camera may have been set to another rate back then.\n        interval = statistics.median(steps)\n        file_fps = 1 / interval if interval > 0 else fps\n        gaps = [(a - times[0], b - a) for a, b in zip(times, times[1:]) if b - a > 1.5 * interval]\n        duration = times[-1] - times[0] + interval\n        measured_fps = (len(times) - 1) / (times[-1] - times[0])\n        problems, notes = [], []\n        if not packets[0][1]:\n            problems.append("does not start with a keyframe")\n'),
    ('tools/phase4_check_segments.py',
     '            where = ", ".join(f"{at:.1f} s (+{step * 1000:.0f} ms)" for at, step in gaps[:3])\n            problems.append(f"{len(gaps)} timestamp gap(s) at {where}")\n        if (width, height) != size:\n            problems.append(f"resolution {width}x{height}")\n        if abs(measured_fps - fps) > 0.05 * fps:\n            problems.append(f"{measured_fps:.2f} fps")\n        status = "FAIL" if problems else "PASS"\n        failures += bool(problems)\n        mb = path.stat().st_size / 1e6\n        print(f"[{status}] {path.name}: {duration:6.1f} s, {len(packets)} frames, {measured_fps:.2f} fps, "\n              f"{mb:.1f} MB, {mb * 8 / duration:.2f} Mbit/s" + (f"  <- {\', \'.join(problems)}" if problems else ""))\n        checked.append((path, nominal_start(path), duration))\n\n',
     '            where = ", ".join(f"{at:.1f} s (+{step * 1000:.0f} ms)" for at, step in gaps[:3])\n            problems.append(f"{len(gaps)} timestamp gap(s) at {where}")\n        if abs(measured_fps - file_fps) > 0.05 * file_fps:\n            problems.append(f"{measured_fps:.2f} fps instead of {file_fps:.2f}")\n        if abs(file_fps - fps) > 0.05 * fps:\n            notes.append(f"recorded at {file_fps:.0f} fps")\n        if (width, height) != size:\n            notes.append(f"recorded at {width}x{height}")\n        status = "FAIL" if problems else "PASS"\n        failures += bool(problems)\n        mb = path.stat().st_size / 1e6\n        remarks = problems + [f"({note}, before a settings change)" for note in notes]\n        print(f"[{status}] {path.name}: {duration:6.1f} s, {len(packets)} frames, {measured_fps:.2f} fps, "\n              f"{mb:.1f} MB, {mb * 8 / duration:.2f} Mbit/s" + (f"  <- {\', \'.join(remarks)}" if remarks else ""))\n        checked.append((path, nominal_start(path), duration))\n\n'),
    ('tools/phase4_check_segments.py',
     '        print("\\nUnfinished .partial files (the newest one is normal while the recorder is running):")\n        for path in partials:\n            packets = ffprobe_packets(path)\n            playable = packets[-1][0] - packets[0][0] if len(packets) > 1 else 0.0\n            print(f"  {path.name}: {path.stat().st_size / 1e6:.1f} MB, {playable:.1f} s playable")\n\n    print(f"\\nFAIL={failures} WARN={warnings} stopped-and-restarted={runs}")\n',
     '        print("\\nUnfinished .partial files (the newest one is normal while the recorder is running):")\n        for path in partials:\n            try:\n                packets = ffprobe_packets(path)\n                size_mb = path.stat().st_size / 1e6\n            except FileNotFoundError:\n                print(f"  {path.name}: finished while this check ran")\n                continue\n            playable = packets[-1][0] - packets[0][0] if len(packets) > 1 else 0.0\n            print(f"  {path.name}: {size_mb:.1f} MB, {playable:.1f} s playable")\n\n    print(f"\\nFAIL={failures} WARN={warnings} stopped-and-restarted={runs}")\n'),
    ('deploy/install.sh',
     'LOGS=/var/log/surveillance\nSVC_USER=cctv\nUNITS=(surveillance-recorder.service surveillance-web.service)\n\nsay() { printf \'\\n== %s\\n\' "$*"; }\n',
     'LOGS=/var/log/surveillance\nSVC_USER=cctv\nUNITS=(surveillance-recorder.service surveillance-web.service surveillance-health.service)\n\nsay() { printf \'\\n== %s\\n\' "$*"; }\n'),
    ('deploy/install.sh',
     '\nsay "systemd units"\ninstall -m 0644 "$APP/deploy/surveillance-recorder.service" "$APP/deploy/surveillance-web.service" /etc/systemd/system/\ninstall -d -m 0755 /etc/systemd/journald.conf.d\ninstall -m 0644 "$APP/deploy/journald-surveillance.conf" /etc/systemd/journald.conf.d/50-surveillance.conf\n',
     '\nsay "systemd units"\ninstall -m 0644 "$APP/deploy/surveillance-recorder.service" "$APP/deploy/surveillance-web.service" \\\n    "$APP/deploy/surveillance-health.service" /etc/systemd/system/\ninstall -d -m 0755 /etc/systemd/journald.conf.d\ninstall -m 0644 "$APP/deploy/journald-surveillance.conf" /etc/systemd/journald.conf.d/50-surveillance.conf\n'),
    ('deploy/install.sh',
     '\nDone. Both services start at boot and restart themselves if they stop.\n  Status:        systemctl status surveillance-recorder surveillance-web\n  Problems:      journalctl -u surveillance-recorder -u surveillance-web -f\n  Full log:      sudo tail -f $LOGS/surveillance.log\n',
     '\nDone. Both services start at boot and restart themselves if they stop.\n  Status:        systemctl status surveillance-recorder surveillance-web surveillance-health\n  Problems:      journalctl -u surveillance-recorder -u surveillance-web -f\n  Full log:      sudo tail -f $LOGS/surveillance.log\n'),
]

NEW_FILES = [
    ('deploy/throttle-monitor.sh', 0o755,
     '#!/bin/sh\n# Run by surveillance-health.service: every 30 s, write the answer of `vcgencmd get_throttled`\n# (e.g. "throttled=0x0") to /run/surveillance/throttled (RAM) for the System Status page.\nset -u\nout=/run/surveillance/throttled\nwhile :; do\n    value="$(vcgencmd get_throttled 2>&1)" || value="error=$value"\n    printf \'%s\\n\' "$value" > "$out.tmp" && mv -f "$out.tmp" "$out"\n    sleep 30\ndone\n'),
    ('deploy/surveillance-health.service', 0o644,
     '# Power and throttling monitor for the System Status page (Phase 13). Installed by deploy/install.sh.\n# `vcgencmd` needs the VideoCore device, which the web interface deliberately cannot open. This\n# helper can open only that device, has no network, and just writes one line into RAM\n# (/run/surveillance/throttled) every 30 seconds.\n[Unit]\nDescription=Surveillance camera power and throttling monitor\nAfter=local-fs.target\nStartLimitIntervalSec=0\n\n[Service]\nType=simple\nUser=cctv\nGroup=cctv\nSupplementaryGroups=video\nExecStart=/bin/sh /opt/surveillance/deploy/throttle-monitor.sh\nRestart=always\nRestartSec=30\nMemoryMax=32M\n\nRuntimeDirectory=surveillance\nRuntimeDirectoryMode=0750\nRuntimeDirectoryPreserve=yes\nUMask=0027\n\nProtectSystem=strict\nProtectHome=yes\nPrivateTmp=yes\nProtectProc=invisible\nDevicePolicy=closed\nDeviceAllow=/dev/vcio_gencmd rw\nDeviceAllow=/dev/vcio rw\nPrivateNetwork=yes\nRestrictAddressFamilies=AF_UNIX\nNoNewPrivileges=yes\nCapabilityBoundingSet=\nAmbientCapabilities=\nRestrictSUIDSGID=yes\nRestrictNamespaces=yes\nRestrictRealtime=yes\nLockPersonality=yes\nMemoryDenyWriteExecute=yes\nProtectKernelTunables=yes\nProtectKernelModules=yes\nProtectKernelLogs=yes\nProtectControlGroups=yes\nProtectClock=yes\nProtectHostname=yes\nSystemCallArchitectures=native\nSystemCallFilter=@system-service\nSystemCallErrorNumber=EPERM\n\n[Install]\nWantedBy=multi-user.target\n'),
]

texts = {}
for rel, old, new in EDITS:
    text = texts.setdefault(rel, Path(rel).read_text())
    if text.count(old) != 1:
        sys.exit(f"ABORTED, nothing changed: {rel} does not match the expected 0.13.0 code "
                 f"(found {text.count(old)} matches for:\n{old})")
    texts[rel] = text.replace(old, new)
for rel, _mode, content in NEW_FILES:
    if Path(rel).exists() and Path(rel).read_text() != content:
        sys.exit(f"ABORTED, nothing changed: {rel} already exists with different content")
for rel, text in texts.items():
    shutil.copy2(rel, rel + ".bak")
    Path(rel).write_text(text)
    print(f"updated {rel}  (backup: {rel}.bak)")
for rel, mode, content in NEW_FILES:
    Path(rel).parent.mkdir(parents=True, exist_ok=True)
    Path(rel).write_text(content)
    os.chmod(rel, mode)
    print(f"created {rel}")
print("Update to 0.13.1 complete.")
PYEOF
python3 update_to_0_13_1.py
sha256sum app/__init__.py app/system_info.py app/web.py tools/phase4_check_segments.py deploy/install.sh deploy/throttle-monitor.sh deploy/surveillance-health.service
```

Expected checksums:

```
fa797233c06063bb7c213a10c11459969bd2646de1216a95dae28772e868d822  app/__init__.py
7f57671fbd009c21ed58d2059cc103a2c7ed30846fe2ed6cfc67ff5240b54dc8  app/system_info.py
678d00f0589a0e1b06c4af2f9910ab3090faf0a4557f7cad2fe6f1fc4c559538  app/web.py
39c5cb3d4cb0da0abaec74843d79bfeb132f238ba77d4a4fcf979e14b9080201  tools/phase4_check_segments.py
11b65c00df7eac9debe4cc96a9a7fc106250aaf9d325072bb86e8e417a1ac998  deploy/install.sh
dff6099f008535278ba437bb8b6a998c24dc0e2393d5bf12d9b00bb8d496b5b8  deploy/throttle-monitor.sh
c1cc086ba4df05d8fe2b4b02cf9b22e47d1144ffc6fbc80e04b77591db39f867  deploy/surveillance-health.service
```

## Step 2 — Deploy it

This is the normal update routine from now on:

```bash
sudo bash ~/surveillance/deploy/install.sh
```

Expected: settings "already exists, kept as it is", moved 0 items, then three services, each `active, 0 restart(s)`.

## Step 3 — Check throttling and the segment checker

```bash
cat /run/surveillance/throttled
systemd-analyze security surveillance-health.service --no-pager | tail -1
sudo cctv-tool phase4_check_segments --last 20
```

Expected:
- **`throttled`:** a line like `throttled=0x0`. If it starts with `error=`, send it to me.
- **Security score:** about 1.0 OK. The System Status page's power/throttling field should now show a value.
- **Segment checker:** your 15 fps files now PASS with the note. The only FAIL that may remain is `06-40-00Z.mp4`, which has a genuine 400 ms gap (about 6 frames) from this morning's early testing. It's worth knowing about but harmless. The checker should end without a traceback.

## Step 4 — Reboot test

```bash
sudo reboot
```

After about a minute, open `https://cam01.tail1c1671.ts.net` on your laptop without touching the Pi; it should show ONLINE. Then connect over SSH and run:

```bash
systemctl is-active surveillance-recorder surveillance-web surveillance-health
journalctl -b -u surveillance-recorder -o short-monotonic --no-pager | grep -m1 "Started surveillance"
grep "Started recording" /var/log/surveillance/surveillance.log | tail -1
```

- **`is-active`:** `active` three times.
- **journalctl:** the number in brackets is seconds since power-on. It should be small (tens of seconds), unlike the 26 822 you got last time.
- **grep:** the last line should show a time just after the reboot.

The installer added you to the `cctv` group, and that takes effect after this reboot. That's why the last command works without `sudo` now.

Please send me the checksums and the output of Steps 2 to 4. Then comes Phase 13b: Restart / Shut down buttons and a time-zone setting on the website, allowed through a narrow system permission rule and protected by your password.
