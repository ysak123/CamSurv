Phase 8, the local web dashboard, is ready to run on the Pi. I tested it here with the fake camera and checked it in a real browser; it hasn't run on your Pi yet.

## What Phase 8 adds

**It's a separate program.** Start it with `python3 -m app.web_main`, alongside the recorder. If the web interface ever crashes, recording carries on.

**The pages:**

| Page | Contents |
|---|---|
| Dashboard | Placeholder for live video (Phase 10); Camera ONLINE / OFFLINE / NOT RUNNING; Recording YES / NO / PAUSED; Motion NONE / DETECTED; storage used vs. limit, with a bar; today's recordings and events; hours of history the limit holds |
| System Status | Everything from your spec: camera, recording and motion status; CPU, RAM, temperature, power/throttling, storage, Wi-Fi (network name and signal), uptime, versions, last recording, last motion; plus clock-sync and database health |
| Storage | Usage, segment count, oldest and newest recording, disk free, average bitrate, hours of history, the limits |
| Settings | Read-only for now. Editing needs the login system from Phase 11 |
| Live View, Recordings, Motion Events | Placeholders until Phases 9–10 |

**Live updates:**
- The page asks for new status every 5 seconds and **only while the tab is visible**, instead of reloading the whole page.
- The recorder writes a small `status.json` every 2 s into a folder held in RAM, so it costs nothing in SD-card writes.
- If that file is more than 10 s old, the web app reports "NOT RUNNING".

**Security, which stays in place for every later phase:**
- **Local only.** It listens only on `127.0.0.1`, and config validation refuses any other address.
- **Host-header check.** Requests must be addressed to `localhost` or `127.0.0.1`. This blocks a malicious website from reaching the camera through your browser (a DNS-rebinding attack).
- **Strict headers.** A Content-Security-Policy with no inline scripts and nothing loaded from other sites, plus anti-framing, no-referrer and `no-store` caching. No `Server` header is sent.
- **No HTML injection.** Templates escape everything, and the JavaScript only ever sets plain text.
- **Read-only access.** The database is opened read-only; this process can't change recordings.

**How you'll reach it:** through an **SSH tunnel** from your laptop. This uses the SSH access you already have, with nothing opened on your network, until Tailscale is set up in Phase 12.

**Also in this update:**
- The web app writes its own log file, `logs/web.log`, because one log file can't be shared between two processes.
- **Database backups are now hourly, keeping 24.** Your step-5 test lost all 6 motion events recorded after the only backup (14:03:58); now at most about an hour could be lost.

## What I tested here

- **Pages:** all of them return 200, unknown pages return 404, and POST requests are refused (405). There are no state-changing routes yet.
- **Host check:** `localhost:8080` and `127.0.0.1:8080` are allowed; `evil.example.com` and `127.0.0.1.nip.io` get 400.
- **Recorder states:** ONLINE while recording, **STOPPED** after Ctrl+C, and **NOT RUNNING** within 10 s of a `kill -9`.
- **In Chrome:** the dashboard filled in live (ONLINE, YES, NONE, storage, "History kept: about 3 d 7 h"). The System Status page showed every field, and the phone layout stacks the cards cleanly.
- **Update script:** produces files identical to the tested ones, and aborts safely if run twice.

## Install

```bash
sudo apt install -y python3-flask python3-waitress
```

## Update the files

Run everything from `~/surveillance`.

**1. Update script** for `__init__.py`, `config.py`, `logging_setup.py`, `database.py` and `recorder.py`:

```bash
cd ~/surveillance
cat > update_to_0_8_0.py <<'EOF'
#!/usr/bin/env python3
"""Update the surveillance project from 0.7.0 to 0.8.0 (run from ~/surveillance)."""
import shutil, sys
from pathlib import Path

EDITS = [
    ('app/__init__.py',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.7.0"\n',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.8.0"\n'),
    ('app/config.py',
     '\nimport json\nimport re\nfrom dataclasses import asdict, dataclass, field, fields\n',
     '\nimport json\nimport os\nimport re\nfrom dataclasses import asdict, dataclass, field, fields\n'),
    ('app/config.py',
     'LOG_FORMATS = ("text", "json")\nRETENTION_MODES = ("oldest_first",)\nMAX_ENCODER_PIXELS = 1920 * 1080\nCAMERA_NAME_PATTERN = re.compile(r"^[\\w][\\w .,\'()-]{0,39}$")\n',
     'LOG_FORMATS = ("text", "json")\nRETENTION_MODES = ("oldest_first",)\nLOCAL_WEB_HOSTS = ("127.0.0.1", "::1", "localhost")\nMAX_ENCODER_PIXELS = 1920 * 1080\nCAMERA_NAME_PATTERN = re.compile(r"^[\\w][\\w .,\'()-]{0,39}$")\n'),
    ('app/config.py',
     '\n@dataclass(frozen=True)\nclass PathSettings:\n    recordings_dir: str = "recordings"\n    database_dir: str = "database"\n    log_dir: str = "logs"\n\n    def validate(self) -> None:\n',
     '\n@dataclass(frozen=True)\nclass WebSettings:\n    host: str = "127.0.0.1"\n    port: int = 8080\n    status_refresh_seconds: float = 5.0\n\n    def validate(self) -> None:\n        if self.host not in LOCAL_WEB_HOSTS:\n            raise ConfigError("web.host must be 127.0.0.1, ::1 or localhost: the web interface is never "\n                              "exposed directly; remote access goes through the private VPN proxy")\n        _check_range("web.port", self.port, 1024, 65535)\n        _check_range("web.status_refresh_seconds", self.status_refresh_seconds, 2.0, 60.0)\n\n\n@dataclass(frozen=True)\nclass PathSettings:\n    recordings_dir: str = "recordings"\n    database_dir: str = "database"\n    log_dir: str = "logs"\n    runtime_dir: str = ""   # live status file; empty = a RAM-backed folder chosen automatically\n\n    def validate(self) -> None:\n'),
    ('app/config.py',
     '            if not value.strip() or "\\x00" in value:\n                raise ConfigError(f"paths.{name} must be a non-empty path")\n\n\n',
     '            if not value.strip() or "\\x00" in value:\n                raise ConfigError(f"paths.{name} must be a non-empty path")\n        if "\\x00" in self.runtime_dir:\n            raise ConfigError("paths.runtime_dir is not a valid path")\n\n\n'),
    ('app/config.py',
     '    storage: StorageSettings = field(default_factory=StorageSettings)\n    motion: MotionSettings = field(default_factory=MotionSettings)\n    paths: PathSettings = field(default_factory=PathSettings)\n    logging: LoggingSettings = field(default_factory=LoggingSettings)\n',
     '    storage: StorageSettings = field(default_factory=StorageSettings)\n    motion: MotionSettings = field(default_factory=MotionSettings)\n    web: WebSettings = field(default_factory=WebSettings)\n    paths: PathSettings = field(default_factory=PathSettings)\n    logging: LoggingSettings = field(default_factory=LoggingSettings)\n'),
    ('app/config.py',
     '    def log_dir(self) -> Path:\n        return resolve_path(self.paths.log_dir)\n\n\n',
     '    def log_dir(self) -> Path:\n        return resolve_path(self.paths.log_dir)\n\n    @property\n    def runtime_dir(self) -> Path:\n        """RAM-backed folder shared by the recorder and web processes (no SD-card writes)."""\n        if self.paths.runtime_dir:\n            return resolve_path(self.paths.runtime_dir)\n        base = os.environ.get("XDG_RUNTIME_DIR")\n        return Path(base) / "surveillance" if base else Path(f"/tmp/surveillance-{os.getuid()}")\n\n\n'),
    ('app/logging_setup.py',
     '\n\ndef setup_logging(settings: LoggingSettings, log_dir: Path) -> Path | None:\n    """Configure the root logger. Returns the log file path, or None if it is not writable."""\n    formatter: logging.Formatter = (JsonFormatter() if settings.format == "json"\n',
     '\n\ndef setup_logging(settings: LoggingSettings, log_dir: Path, file_name: str = LOG_FILE_NAME) -> Path | None:\n    """Configure the root logger. Returns the log file path, or None if it is not writable."""\n    formatter: logging.Formatter = (JsonFormatter() if settings.format == "json"\n'),
    ('app/logging_setup.py',
     '        root.addHandler(console)\n\n    log_file: Path | None = log_dir / LOG_FILE_NAME\n    try:\n        log_dir.mkdir(parents=True, exist_ok=True)\n',
     '        root.addHandler(console)\n\n    # Each process gets its own file: RotatingFileHandler cannot be shared between processes.\n    log_file: Path | None = log_dir / file_name\n    try:\n        log_dir.mkdir(parents=True, exist_ok=True)\n'),
    ('app/database.py',
     '  * At startup: integrity check; a damaged file is moved aside and the newest good backup\n    is restored (or a fresh database is created), then the index is rebuilt from the files.\n  * A backup copy is made once a day (VACUUM INTO), keeping the last few.\n  * Any database error is logged and recording carries on.\n\n',
     '  * At startup: integrity check; a damaged file is moved aside and the newest good backup\n    is restored (or a fresh database is created), then the index is rebuilt from the files.\n  * A backup copy is made every hour (VACUUM INTO), keeping the last 24; motion events\n    exist only here, so frequent backups limit what a damaged file can lose.\n  * Any database error is logged and recording carries on.\n\n'),
    ('app/database.py',
     'DB_FILE_NAME = "surveillance.db"\nBACKUP_DIR_NAME = "backups"\nBACKUPS_KEPT = 3\nBACKUP_INTERVAL_S = 24 * 3600\nPRUNE_INTERVAL_S = 6 * 3600\nIDLE_WAKE_S = 60.0\n',
     'DB_FILE_NAME = "surveillance.db"\nBACKUP_DIR_NAME = "backups"\nBACKUPS_KEPT = 24\nBACKUP_INTERVAL_S = 3600\nPRUNE_INTERVAL_S = 6 * 3600\nIDLE_WAKE_S = 60.0\n'),
    ('app/recorder.py',
     '        return self._output.write_error if self._output else None\n\n    def _boundary_loop(self, encoder: H264Encoder) -> None:\n        """Request a keyframe at each boundary so segments start on time, not up to 2 s late."""\n',
     '        return self._output.write_error if self._output else None\n\n    @property\n    def paused(self) -> bool:\n        return bool(self._output and self._output.paused)\n\n    def _boundary_loop(self, encoder: H264Encoder) -> None:\n        """Request a keyframe at each boundary so segments start on time, not up to 2 s late."""\n'),
]

texts = {}
for rel, old, new in EDITS:
    text = texts.setdefault(rel, Path(rel).read_text())
    if text.count(old) != 1:
        sys.exit(f"ABORTED, nothing changed: {rel} does not match the expected 0.7.0 code "
                 f"(found {text.count(old)} matches for:\n{old})")
    texts[rel] = text.replace(old, new)
for rel, text in texts.items():
    shutil.copy2(rel, rel + ".bak")
    Path(rel).write_text(text)
    print(f"updated {rel}  (backup: {rel}.bak)")
print("Update to 0.8.0 complete.")
EOF
python3 update_to_0_8_0.py
```

**2. Replace `app/main.py`** (it now publishes the live status):

```bash
cat > ~/surveillance/app/main.py <<'EOF'
"""Recorder service entry point.

Run from the project directory:   python3 -m app.main [--config PATH]

Supervises the recording pipeline: if the camera cannot be opened, or frames stop arriving,
the pipeline is torn down and restarted with exponential backoff (5 s up to 60 s).
SIGTERM/SIGINT (Ctrl+C) stop cleanly and finalise the current segment.
"""
from __future__ import annotations

import os

# libcamera reads this when it is first loaded, so it must be set before app.recorder is imported.
os.environ.setdefault("LIBCAMERA_LOG_LEVELS", "*:WARN")

import argparse  # noqa: E402
import logging  # noqa: E402
import signal  # noqa: E402
import sys  # noqa: E402
import threading  # noqa: E402
import time  # noqa: E402
from pathlib import Path  # noqa: E402

from app import __version__  # noqa: E402
from app.camera import CameraError  # noqa: E402
from app.clock import ClockMonitor  # noqa: E402
from app.config import DEFAULT_CONFIG_PATH, ConfigError, Settings, load_settings  # noqa: E402
from app.database import Database  # noqa: E402
from app.logging_setup import setup_logging  # noqa: E402
from app.recorder import Recorder  # noqa: E402
from app.status import RecorderState, StatusPublisher  # noqa: E402
from app.storage_manager import StorageManager  # noqa: E402

log = logging.getLogger("Main")

BACKOFF_INITIAL_S = 5
BACKOFF_MAX_S = 60
HEALTHY_RESET_S = 120


def parse_args(argv: list[str] | None = None) -> argparse.Namespace:
    parser = argparse.ArgumentParser(description="Raspberry Pi surveillance recorder")
    parser.add_argument("--config", type=Path,
                        default=Path(os.environ.get("SURVEILLANCE_CONFIG", DEFAULT_CONFIG_PATH)),
                        help=f"settings file (default {DEFAULT_CONFIG_PATH})")
    return parser.parse_args(argv)


def open_database(settings: Settings) -> Database | None:
    db = Database(settings)
    try:
        db.open()
    except Exception:  # noqa: BLE001 - recording must work even without the index
        log.exception("Cannot open the database; recording continues without an index")
        return None
    db.start()
    return db


def run(settings: Settings, stop: threading.Event) -> None:
    clock = ClockMonitor()
    clock.start()
    db = open_database(settings)
    active: list[Recorder] = []
    storage = StorageManager(settings, lambda: active[0].current_partial_path if active else None,
                             on_deleted=db.recordings_deleted if db else None,
                             on_recovered=db.recording_recovered if db else None)
    state = RecorderState()
    publisher = StatusPublisher(settings, state, clock, storage, db, lambda: active[0] if active else None)
    publisher.start()
    # Recover unfinished segments and free space before the camera starts writing.
    try:
        storage.run_cycle()
    except Exception:  # noqa: BLE001 - the periodic check retries; recording matters more
        log.exception("Initial storage check failed")
    storage.start()
    if db:
        db.reconcile()
    backoff = BACKOFF_INITIAL_S
    try:
        while not stop.is_set():
            recorder = Recorder(settings, clock, storage.can_write, storage.location_error)
            recorder.add_listener(lambda _segment: storage.trigger())
            recorder.add_listener(publisher.on_segment)
            recorder.add_motion_listener(publisher.on_motion)
            if db:
                recorder.add_listener(db.add_segment)
                recorder.add_motion_listener(db.motion_event)
            active[:] = [recorder]
            state.set("starting")
            try:
                recorder.start()
            except (CameraError, OSError) as exc:
                log.error("Could not start recording: %s. Retrying in %d s", exc, backoff)
                state.set("retrying", str(exc))
                recorder.stop()
                stop.wait(backoff)
                backoff = min(backoff * 2, BACKOFF_MAX_S)
                continue

            state.set("recording")
            started = time.monotonic()
            while not stop.wait(1.0):
                problem = recorder.health()
                if problem:
                    log.error("Recording unhealthy: %s. Restarting the camera pipeline", problem)
                    state.set("retrying", problem)
                    break
                if time.monotonic() - started > HEALTHY_RESET_S:
                    backoff = BACKOFF_INITIAL_S
            if stop.is_set():
                state.set("stopping")
            recorder.stop()
            if not stop.is_set():
                stop.wait(backoff)
                backoff = min(backoff * 2, BACKOFF_MAX_S)
    finally:
        active.clear()
        state.set("stopped")
        publisher.stop()
        storage.stop()
        if db:
            db.stop()
        clock.stop()


def main(argv: list[str] | None = None) -> int:
    args = parse_args(argv)
    try:
        settings, created = load_settings(args.config)
    except ConfigError as exc:
        print(f"Configuration error: {exc}", file=sys.stderr)
        return 2

    log_file = setup_logging(settings.logging, settings.log_dir)
    log.info("Surveillance recorder %s starting (config %s%s, log %s)", __version__, args.config,
             ", created with defaults" if created else "", log_file or "console only")

    stop = threading.Event()

    def request_stop(signum, _frame) -> None:
        if not stop.is_set():
            log.info("Received %s, stopping", signal.Signals(signum).name)
        stop.set()

    signal.signal(signal.SIGTERM, request_stop)
    signal.signal(signal.SIGINT, request_stop)

    try:
        run(settings, stop)
    except Exception:  # noqa: BLE001 - log it; systemd restarts the service
        log.exception("Fatal error")
        return 1
    log.info("Surveillance recorder stopped")
    return 0


if __name__ == "__main__":
    sys.exit(main())
EOF
```

**3. New files in `app/`:**

```bash
cat > ~/surveillance/app/status.py <<'EOF'
"""Live recorder status shared with the web interface through a small JSON file.

The recorder rewrites  <runtime_dir>/status.json  every 2 seconds. runtime_dir lives in RAM
(tmpfs), so this costs no SD-card writes. The web process only reads the file; if it has not
been updated for STALE_AFTER_S seconds the recorder is reported as not running.
"""
from __future__ import annotations

import json
import logging
import os
import threading
import time
from pathlib import Path
from typing import Any, Callable

from app import __version__
from app.fileutil import atomic_write_bytes

log = logging.getLogger("Status")

STATUS_FILE_NAME = "status.json"
PUBLISH_INTERVAL_S = 2.0
STALE_AFTER_S = 10.0


def now_ms() -> int:
    return int(time.time() * 1000)


class RecorderState:
    """What the supervisor in main.py is doing right now."""

    def __init__(self) -> None:
        self.phase = "starting"      # starting | recording | retrying | stopping | stopped
        self.message = ""

    def set(self, phase: str, message: str = "") -> None:
        self.phase = phase
        self.message = message


class StatusPublisher:
    def __init__(self, settings, state: RecorderState, clock, storage, db,
                 get_recorder: Callable[[], Any]) -> None:
        self._path = settings.runtime_dir / STATUS_FILE_NAME
        self._settings = settings
        self._state = state
        self._clock = clock
        self._storage = storage
        self._db = db
        self._get_recorder = get_recorder
        self._started_at = now_ms()
        self._last_segment: dict | None = None
        self._motion_active = False
        self._last_motion_start: int | None = None
        self._stop = threading.Event()
        self._thread = threading.Thread(target=self._run, name="status-publisher", daemon=True)
        self._write_failed = False

    # Recorder listeners
    def on_segment(self, segment) -> None:
        self._last_segment = {"rel_path": segment.rel_path, "end_utc": int(segment.end_utc.timestamp() * 1000),
                              "has_motion": segment.has_motion}

    def on_motion(self, kind: str, event) -> None:
        self._motion_active = kind == "start"
        if kind == "start":
            self._last_motion_start = int(event.start * 1000)

    def start(self) -> None:
        self._path.parent.mkdir(parents=True, exist_ok=True, mode=0o700)
        self._thread.start()

    def stop(self) -> None:
        self._stop.set()
        if self._thread.is_alive():
            self._thread.join(timeout=5)
        self.publish()

    def _run(self) -> None:
        while not self._stop.wait(PUBLISH_INTERVAL_S):
            self.publish()

    def snapshot(self) -> dict:
        recorder = self._get_recorder()
        storage = self._storage.status if self._storage else None
        cam = self._settings.camera
        recording = self._state.phase == "recording" and recorder is not None
        return {
            "version": __version__,
            "pid": os.getpid(),
            "updated_at": now_ms(),
            "started_at": self._started_at,
            "recorder": {"phase": self._state.phase, "message": self._state.message},
            "camera": {"name": cam.name, "resolution": f"{cam.width}x{cam.height}", "fps": cam.framerate,
                       "bitrate": cam.bitrate},
            "recording": {
                "active": recording and not recorder.paused,
                "paused": bool(recording and recorder.paused),
                "current_segment": recorder.current_segment if recording else None,
                "write_error": recorder.write_error if recording else None,
                "last_segment": self._last_segment,
            },
            "motion": {"enabled": self._settings.motion.enabled, "active": self._motion_active and recording,
                       "last_event_start": self._last_motion_start},
            "storage": None if storage is None else {
                "state": storage.state, "message": storage.message, "used_bytes": storage.used_bytes,
                "free_bytes": storage.free_bytes, "total_bytes": storage.total_bytes,
                "max_bytes": storage.max_bytes, "min_free_bytes": storage.min_free_bytes,
                "segment_count": storage.segment_count,
                "oldest_start": int(storage.oldest_start * 1000) if storage.oldest_start else None,
                "newest_start": int(storage.newest_start * 1000) if storage.newest_start else None},
            "clock_synced": self._clock.synchronized,
            "database_healthy": self._db.healthy if self._db else None,
        }

    def publish(self) -> None:
        try:
            atomic_write_bytes(self._path, json.dumps(self.snapshot()).encode(), mode=0o600)
            self._write_failed = False
        except Exception as exc:  # noqa: BLE001 - status is informational; never disturb recording
            if not self._write_failed:
                log.error("Cannot write the status file %s: %s", self._path, exc)
            self._write_failed = True


def read_status(runtime_dir: Path) -> dict | None:
    """The recorder's latest status, or None if it is not running (file missing or stale)."""
    try:
        data = json.loads((runtime_dir / STATUS_FILE_NAME).read_text())
    except (OSError, ValueError):
        return None
    if not isinstance(data, dict):
        return None
    phase = (data.get("recorder") or {}).get("phase")
    if now_ms() - data.get("updated_at", 0) > STALE_AFTER_S * 1000 and phase != "stopped":
        return None
    return data
EOF
```

```bash
cat > ~/surveillance/app/system_info.py <<'EOF'
"""Raspberry Pi health figures read straight from the kernel (no extra packages)."""
from __future__ import annotations

import os
import shutil
import socket
import subprocess
import threading
import time
from pathlib import Path

CACHE_S = 30.0

THROTTLE_FLAGS = {
    0: "under-voltage now",
    1: "frequency capped now",
    2: "throttled now",
    3: "soft temperature limit now",
    16: "under-voltage since boot",
    17: "frequency capped since boot",
    18: "throttled since boot",
    19: "soft temperature limit since boot",
}


def _read(path: str) -> str | None:
    try:
        return Path(path).read_text().strip()
    except OSError:
        return None


def _run(args: list[str]) -> str | None:
    if shutil.which(args[0]) is None:
        return None
    try:
        result = subprocess.run(args, capture_output=True, text=True, timeout=5, check=False)
    except (OSError, subprocess.TimeoutExpired):
        return None
    return result.stdout.strip() if result.returncode == 0 else None


class SystemInfo:
    """Thread-safe; CPU usage is measured between successive calls."""

    def __init__(self) -> None:
        self._lock = threading.Lock()
        self._last_cpu: tuple[int, int] | None = None
        self._last_cpu_percent: float | None = None
        self._cache: dict[str, tuple[float, object]] = {}

    def snapshot(self) -> dict:
        return {
            "hostname": socket.gethostname(),
            "cpu_percent": self.cpu_percent(),
            "cpu_count": os.cpu_count(),
            "load": [round(x, 2) for x in os.getloadavg()],
            "memory": self.memory(),
            "temperature_c": self.temperature(),
            "throttled": self._cached("throttled", self.throttled),
            "uptime_s": self.uptime(),
            "network": self._cached("network", self.network),
        }

    def _cached(self, key: str, fn):
        with self._lock:
            hit = self._cache.get(key)
            if hit and time.monotonic() - hit[0] < CACHE_S:
                return hit[1]
        value = fn()
        with self._lock:
            self._cache[key] = (time.monotonic(), value)
        return value

    def cpu_percent(self) -> float | None:
        line = (_read("/proc/stat") or "").splitlines()
        if not line:
            return None
        values = [int(v) for v in line[0].split()[1:]]
        idle = values[3] + values[4]   # idle + iowait
        total = sum(values)
        with self._lock:
            previous, self._last_cpu = self._last_cpu, (total, idle)
            if previous and total > previous[0]:
                busy = 1 - (idle - previous[1]) / (total - previous[0])
                self._last_cpu_percent = round(max(0.0, min(1.0, busy)) * 100, 1)
            return self._last_cpu_percent

    @staticmethod
    def memory() -> dict | None:
        fields = {}
        for line in (_read("/proc/meminfo") or "").splitlines():
            key, _, rest = line.partition(":")
            fields[key] = int(rest.split()[0]) * 1024 if rest.split() else 0
        if "MemTotal" not in fields:
            return None
        total, available = fields["MemTotal"], fields.get("MemAvailable", 0)
        return {"total_bytes": total, "used_bytes": total - available,
                "used_percent": round((total - available) / total * 100, 1)}

    @staticmethod
    def temperature() -> float | None:
        raw = _read("/sys/class/thermal/thermal_zone0/temp")
        return round(int(raw) / 1000, 1) if raw and raw.isdigit() else None

    @staticmethod
    def throttled() -> dict | None:
        out = _run(["vcgencmd", "get_throttled"])
        if not out or "=" not in out:
            return None
        try:
            value = int(out.split("=", 1)[1], 16)
        except ValueError:
            return None
        return {"raw": f"0x{value:x}", "flags": [text for bit, text in THROTTLE_FLAGS.items() if value & (1 << bit)]}

    @staticmethod
    def uptime() -> int | None:
        raw = _read("/proc/uptime")
        return int(float(raw.split()[0])) if raw else None

    @staticmethod
    def network() -> dict:
        wifi = None
        for line in (_read("/proc/net/wireless") or "").splitlines()[2:]:
            name, _, rest = line.partition(":")
            parts = rest.split()
            if len(parts) >= 3:
                wifi = {"interface": name.strip(), "signal_dbm": int(float(parts[2].rstrip(".")))}
                break
        if wifi:
            wifi["state"] = _read(f"/sys/class/net/{wifi['interface']}/operstate")
            nm = _run(["nmcli", "-t", "-f", "ACTIVE,SSID", "dev", "wifi"])
            ssid = next((line.split(":", 1)[1] for line in (nm or "").splitlines() if line.startswith("yes:")), None)
            if ssid is None:
                iw = _run(["iw", "dev", wifi["interface"], "link"]) or ""
                ssid = next((line.split("SSID:", 1)[1].strip() for line in iw.splitlines() if "SSID:" in line), None)
            wifi["ssid"] = ssid
        wired = _read("/sys/class/net/eth0/operstate")
        return {"wifi": wifi, "ethernet": wired}
EOF
```

```bash
cat > ~/surveillance/app/web.py <<'EOF'
"""Web interface (Flask). Phase 8: read-only dashboard, system status, storage and settings pages.

Security baseline, kept in every later phase:
  * listens on 127.0.0.1 only (enforced by config validation)
  * Host header allow-list, which defeats DNS-rebinding attacks from other websites
  * strict Content-Security-Policy: no inline scripts, no external resources
  * templates auto-escape everything; the JavaScript only ever sets textContent
  * the database is opened read-only; this process cannot change recordings
"""
from __future__ import annotations

import sqlite3
import time
from dataclasses import asdict
from datetime import datetime

from flask import Flask, abort, jsonify, render_template, request

from app import __version__
from app.config import PROJECT_ROOT, Settings
from app.database import DB_FILE_NAME, connect
from app.status import read_status
from app.system_info import SystemInfo

TEMPLATE_DIR = PROJECT_ROOT / "web" / "templates"
STATIC_DIR = PROJECT_ROOT / "web" / "static"
ALLOWED_HOSTNAMES = {"127.0.0.1", "localhost", "[::1]"}
BITRATE_SAMPLE = 12

SECURITY_HEADERS = {
    "Content-Security-Policy": ("default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' data:; "
                                "media-src 'self'; connect-src 'self'; object-src 'none'; base-uri 'none'; "
                                "frame-ancestors 'none'; form-action 'self'"),
    "X-Content-Type-Options": "nosniff",
    "X-Frame-Options": "DENY",
    "Referrer-Policy": "no-referrer",
    "Permissions-Policy": "camera=(), microphone=(), geolocation=(), payment=(), usb=()",
    "Cross-Origin-Opener-Policy": "same-origin",
    "Cross-Origin-Resource-Policy": "same-origin",
}

NAVIGATION = [
    ("dashboard", "Dashboard"),
    ("live", "Live View"),
    ("recordings", "Recordings"),
    ("events", "Motion Events"),
    ("storage", "Storage"),
    ("settings_page", "Settings"),
    ("system", "System Status"),
]


def day_start_ms() -> int:
    now = datetime.now().astimezone()
    return int(now.replace(hour=0, minute=0, second=0, microsecond=0).timestamp() * 1000)


def database_summary(settings: Settings) -> dict:
    path = settings.database_dir / DB_FILE_NAME
    if not path.exists():
        return {"available": False}
    try:
        conn = connect(path, read_only=True)
        try:
            today = day_start_ms()
            row = conn.execute(
                "SELECT (SELECT MAX(end_utc) FROM recordings) AS last_recording_end,"
                " (SELECT MAX(start_utc) FROM motion_events) AS last_motion_start,"
                " (SELECT COUNT(*) FROM recordings WHERE start_utc >= ?) AS recordings_today,"
                " (SELECT COUNT(*) FROM motion_events WHERE start_utc >= ?) AS events_today",
                (today, today)).fetchone()
            sample = conn.execute(
                "SELECT size_bytes, duration_ms FROM recordings WHERE status = 'complete' AND duration_ms >= ?"
                " ORDER BY start_utc DESC LIMIT ?",
                (int(settings.recording.segment_seconds * 1000 * 0.95), BITRATE_SAMPLE)).fetchall()
        finally:
            conn.close()
    except sqlite3.Error as exc:
        return {"available": False, "error": str(exc)}
    bytes_per_s = (sum(r["size_bytes"] for r in sample) / (sum(r["duration_ms"] for r in sample) / 1000)
                   if sample else None)
    return {"available": True, **dict(row), "bytes_per_second": bytes_per_s}


def create_app(settings: Settings) -> Flask:
    app = Flask(__name__, template_folder=str(TEMPLATE_DIR), static_folder=str(STATIC_DIR))
    app.config.update(JSON_SORT_KEYS=False, MAX_CONTENT_LENGTH=64 * 1024)
    system_info = SystemInfo()

    @app.before_request
    def check_host() -> None:
        hostname = request.host.rsplit(":", 1)[0] if not request.host.startswith("[") \
            else request.host.split("]")[0] + "]"
        if hostname not in ALLOWED_HOSTNAMES:
            abort(400)

    @app.after_request
    def security_headers(response):
        for name, value in SECURITY_HEADERS.items():
            response.headers.setdefault(name, value)
        if not request.path.startswith("/static/"):
            response.headers["Cache-Control"] = "no-store"
        response.headers.pop("Server", None)
        return response

    @app.context_processor
    def common() -> dict:
        return {"camera_name": settings.camera.name, "navigation": NAVIGATION, "version": __version__,
                "refresh_ms": int(settings.web.status_refresh_seconds * 1000)}

    @app.get("/")
    def dashboard():
        return render_template("dashboard.html", title="Dashboard", page="dashboard")

    @app.get("/system")
    def system():
        return render_template("system.html", title="System Status", page="system")

    @app.get("/storage")
    def storage():
        return render_template("storage.html", title="Storage", page="storage", storage=settings.storage,
                               recordings_dir=settings.recordings_dir)

    @app.get("/settings")
    def settings_page():
        return render_template("settings.html", title="Settings", page="settings_page", sections=asdict(settings))

    @app.get("/live")
    def live():
        return render_template("placeholder.html", title="Live View", page="live", phase=10,
                               text="Live view is added in Phase 10.")

    @app.get("/recordings")
    def recordings():
        return render_template("placeholder.html", title="Recordings", page="recordings", phase=9,
                               text="Browsing, searching and playing recordings is added in Phase 9.")

    @app.get("/events")
    def events():
        return render_template("placeholder.html", title="Motion Events", page="events", phase=9,
                               text="The motion event timeline is added in Phase 9.")

    @app.get("/api/status")
    def api_status():
        return jsonify({
            "server_time": int(time.time() * 1000),
            "web_version": __version__,
            "recorder": read_status(settings.runtime_dir),
            "system": system_info.snapshot(),
            "database": database_summary(settings),
        })

    @app.errorhandler(400)
    def bad_request(_error):
        return "Bad request", 400

    @app.errorhandler(404)
    def not_found(_error):
        return render_template("placeholder.html", title="Not found", page="", phase=None,
                               text="This page does not exist."), 404

    return app
EOF
```

```bash
cat > ~/surveillance/app/web_main.py <<'EOF'
"""Web service entry point.

Run from the project directory:   python3 -m app.web_main [--config PATH]

Serves the dashboard on 127.0.0.1 only. From your computer, open an SSH tunnel:
    ssh -L 8080:127.0.0.1:8080 ysak@ysak.local
then browse to http://localhost:8080
"""
from __future__ import annotations

import argparse
import logging
import os
import sys
from pathlib import Path

from waitress import serve

from app import __version__
from app.config import DEFAULT_CONFIG_PATH, ConfigError, load_settings
from app.logging_setup import setup_logging
from app.web import create_app

log = logging.getLogger("Web")

WEB_LOG_FILE = "web.log"
WORKER_THREADS = 6


def main(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser(description="Surveillance web interface")
    parser.add_argument("--config", type=Path,
                        default=Path(os.environ.get("SURVEILLANCE_CONFIG", DEFAULT_CONFIG_PATH)))
    args = parser.parse_args(argv)
    try:
        settings, _ = load_settings(args.config, create_if_missing=False)
    except ConfigError as exc:
        print(f"Configuration error: {exc}", file=sys.stderr)
        return 2

    setup_logging(settings.logging, settings.log_dir, WEB_LOG_FILE)
    logging.getLogger("waitress").setLevel(logging.WARNING)
    host, port = settings.web.host, settings.web.port
    log.info("Web interface %s listening on http://%s:%d (local only)", __version__, host, port)
    serve(create_app(settings), host=host, port=port, threads=WORKER_THREADS, ident="",
          clear_untrusted_proxy_headers=True)
    return 0


if __name__ == "__main__":
    sys.exit(main())
EOF
```

**4. Templates and static files:**

```bash
mkdir -p ~/surveillance/web/templates ~/surveillance/web/static

cat > ~/surveillance/web/templates/base.html <<'EOF'
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <meta name="referrer" content="no-referrer">
  <title>{{ title }} · {{ camera_name }}</title>
  <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
  <script src="{{ url_for('static', filename='app.js') }}" defer></script>
</head>
<body data-refresh="{{ refresh_ms }}">
  <header class="topbar">
    <div class="brand">
      <span class="dot" id="header-dot" data-level="unknown" aria-hidden="true"></span>
      <div>
        <h1>Raspberry Pi Security Camera</h1>
        <p class="camera-name">{{ camera_name }}</p>
      </div>
    </div>
    <p class="clock" id="updated-at" aria-live="polite">connecting…</p>
  </header>
  <nav class="nav" aria-label="Main">
    {% for endpoint, label in navigation %}
      <a href="{{ url_for(endpoint) }}" {% if page == endpoint %}class="active" aria-current="page"{% endif %}>{{ label }}</a>
    {% endfor %}
  </nav>
  <div class="banner" id="connection-banner" role="alert" hidden>Cannot reach the camera. Retrying…</div>
  <main>
    {% block content %}{% endblock %}
  </main>
  <footer class="footer">Surveillance {{ version }} · local interface</footer>
</body>
</html>
EOF

cat > ~/surveillance/web/templates/dashboard.html <<'EOF'
{% extends "base.html" %}
{% block content %}
<section class="live-box" aria-label="Live camera">
  <div class="live-placeholder">
    <span class="live-title">LIVE CAMERA</span>
    <span class="muted">Live view is added in Phase 10</span>
  </div>
</section>

<section class="cards">
  <article class="card">
    <h2>Camera status</h2>
    <p class="value" id="camera-state">…</p>
    <p class="sub" id="camera-message"></p>
  </article>
  <article class="card">
    <h2>Recording</h2>
    <p class="value" id="recording-state">…</p>
    <p class="sub" id="current-segment"></p>
  </article>
  <article class="card">
    <h2>Motion</h2>
    <p class="value" id="motion-state">…</p>
    <p class="sub">Last detected: <span id="last-motion">…</span></p>
  </article>
  <article class="card">
    <h2>Storage</h2>
    <p class="value" id="storage-summary">…</p>
    <progress id="storage-bar" max="100" value="0" aria-label="Storage used"></progress>
    <p class="sub" id="storage-message"></p>
  </article>
</section>

<section class="panel">
  <h2>Today</h2>
  <dl class="facts">
    <div><dt>Recordings today</dt><dd id="recordings-today">…</dd></div>
    <div><dt>Motion events today</dt><dd id="events-today">…</dd></div>
    <div><dt>Last completed recording</dt><dd id="last-recording">…</dd></div>
    <div><dt>History kept</dt><dd id="retention">…</dd></div>
  </dl>
</section>
{% endblock %}
EOF

cat > ~/surveillance/web/templates/system.html <<'EOF'
{% extends "base.html" %}
{% block content %}
<section class="panel">
  <h2>Camera and recording</h2>
  <dl class="facts">
    <div><dt>Camera status</dt><dd id="camera-state">…</dd></div>
    <div><dt>Recording status</dt><dd id="recording-state">…</dd></div>
    <div><dt>Motion detection</dt><dd id="motion-state">…</dd></div>
    <div><dt>Current segment</dt><dd id="current-segment">…</dd></div>
    <div><dt>Last successful recording</dt><dd id="last-recording">…</dd></div>
    <div><dt>Last detected motion</dt><dd id="last-motion">…</dd></div>
    <div><dt>Recorder running for</dt><dd id="recorder-uptime">…</dd></div>
    <div><dt>Camera settings</dt><dd id="camera-config">…</dd></div>
  </dl>
</section>

<section class="panel">
  <h2>Raspberry Pi</h2>
  <dl class="facts">
    <div><dt>CPU usage</dt><dd id="cpu">…</dd></div>
    <div><dt>RAM usage</dt><dd id="memory">…</dd></div>
    <div><dt>CPU temperature</dt><dd id="temperature">…</dd></div>
    <div><dt>Power / throttling</dt><dd id="throttled">…</dd></div>
    <div><dt>Storage</dt><dd id="storage-summary">…</dd></div>
    <div><dt>Wi-Fi</dt><dd id="wifi">…</dd></div>
    <div><dt>Uptime</dt><dd id="uptime">…</dd></div>
    <div><dt>Clock</dt><dd id="clock-sync">…</dd></div>
    <div><dt>Database</dt><dd id="database">…</dd></div>
    <div><dt>Application version</dt><dd id="app-version">…</dd></div>
    <div><dt>Host name</dt><dd id="hostname">…</dd></div>
  </dl>
</section>
{% endblock %}
EOF

cat > ~/surveillance/web/templates/storage.html <<'EOF'
{% extends "base.html" %}
{% block content %}
<section class="panel">
  <h2>Recording storage</h2>
  <p class="value" id="storage-summary">…</p>
  <progress id="storage-bar" max="100" value="0" aria-label="Storage used"></progress>
  <p class="sub" id="storage-message"></p>
  <dl class="facts">
    <div><dt>Recordings folder</dt><dd>{{ recordings_dir }}</dd></div>
    <div><dt>Segments stored</dt><dd id="segment-count">…</dd></div>
    <div><dt>Oldest recording</dt><dd id="oldest">…</dd></div>
    <div><dt>Newest recording</dt><dd id="newest">…</dd></div>
    <div><dt>Disk free</dt><dd id="disk-free">…</dd></div>
    <div><dt>Average bitrate</dt><dd id="bitrate">…</dd></div>
    <div><dt>History kept at this bitrate</dt><dd id="retention">…</dd></div>
  </dl>
</section>

<section class="panel">
  <h2>Limits</h2>
  <dl class="facts">
    <div><dt>Maximum recording storage</dt><dd>{{ storage.max_storage_gb }} GB</dd></div>
    <div><dt>Emergency minimum free space</dt><dd>{{ storage.min_free_gb }} GB</dd></div>
    <div><dt>Maximum age</dt><dd>{% if storage.max_age_days %}{{ storage.max_age_days }} days{% else %}off{% endif %}</dd></div>
    <div><dt>Retention</dt><dd>{{ storage.retention }}</dd></div>
    <div><dt>Required mount</dt><dd>{{ storage.required_mount or "none" }}</dd></div>
  </dl>
  <p class="muted">Changing limits from the browser is added with the login system in Phase 11.
    For now use <code>tools/set_setting.py</code> and restart the recorder.</p>
</section>
{% endblock %}
EOF

cat > ~/surveillance/web/templates/settings.html <<'EOF'
{% extends "base.html" %}
{% block content %}
<p class="notice">Read-only for now: editing settings from the browser is added together with the login system
  in Phase 11. Until then use <code>python3 tools/set_setting.py section.name value</code> and restart the recorder.</p>
<div class="settings-grid">
  {% for section, values in sections.items() %}
  <section class="panel">
    <h2>{{ section|capitalize }}</h2>
    <table class="kv">
      {% for name, value in values.items() %}
      <tr><th scope="row">{{ name }}</th><td>{{ value }}</td></tr>
      {% endfor %}
    </table>
  </section>
  {% endfor %}
</div>
{% endblock %}
EOF

cat > ~/surveillance/web/templates/placeholder.html <<'EOF'
{% extends "base.html" %}
{% block content %}
<section class="panel placeholder">
  <h2>{{ title }}</h2>
  <p>{{ text }}</p>
</section>
{% endblock %}
EOF
```

```bash
cat > ~/surveillance/web/static/style.css <<'EOF'
:root {
  color-scheme: dark;
  --bg: #0f1216;
  --panel: #171b21;
  --panel-2: #1e232b;
  --border: #2a313b;
  --text: #e6e9ee;
  --muted: #8b95a3;
  --accent: #4f9cf9;
  --ok: #3ecf8e;
  --warn: #f5b841;
  --bad: #f25f5c;
  --radius: 12px;
  font-family: system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
}

* { box-sizing: border-box; }

body {
  margin: 0;
  background: var(--bg);
  color: var(--text);
  line-height: 1.45;
}

.topbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  padding: 1rem 1.25rem;
  border-bottom: 1px solid var(--border);
  background: var(--panel);
}

.brand { display: flex; align-items: center; gap: .75rem; }
.brand h1 { margin: 0; font-size: 1.15rem; font-weight: 650; }
.camera-name { margin: 0; color: var(--muted); font-size: .85rem; }
.clock { margin: 0; color: var(--muted); font-size: .8rem; text-align: right; }

.dot {
  width: .8rem;
  height: .8rem;
  border-radius: 50%;
  background: var(--muted);
  flex: none;
}
.dot[data-level="ok"] { background: var(--ok); box-shadow: 0 0 .6rem var(--ok); }
.dot[data-level="warn"] { background: var(--warn); }
.dot[data-level="bad"] { background: var(--bad); }

.nav {
  display: flex;
  gap: .25rem;
  overflow-x: auto;
  padding: .5rem 1rem;
  border-bottom: 1px solid var(--border);
  background: var(--panel);
  scrollbar-width: none;
}
.nav a {
  flex: none;
  padding: .45rem .8rem;
  border-radius: 8px;
  color: var(--muted);
  text-decoration: none;
  font-size: .9rem;
}
.nav a:hover { color: var(--text); background: var(--panel-2); }
.nav a.active { color: var(--text); background: var(--panel-2); box-shadow: inset 0 -2px 0 var(--accent); }

main {
  max-width: 1100px;
  margin: 0 auto;
  padding: 1.25rem;
  display: grid;
  gap: 1.25rem;
}

.banner {
  margin: 0;
  padding: .6rem 1.25rem;
  background: #3a1d1d;
  color: #ffd6d5;
  border-bottom: 1px solid var(--bad);
  font-size: .9rem;
}

.live-box {
  aspect-ratio: 4 / 3;
  max-height: 60vh;
  width: 100%;
  border-radius: var(--radius);
  border: 1px solid var(--border);
  background: repeating-linear-gradient(45deg, #13171c, #13171c 12px, #161a20 12px, #161a20 24px);
  display: grid;
  place-items: center;
}
.live-placeholder { display: grid; gap: .35rem; text-align: center; }
.live-title { font-size: 1.4rem; letter-spacing: .2em; color: var(--muted); }

.cards {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(210px, 1fr));
  gap: 1rem;
}

.card, .panel {
  background: var(--panel);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 1rem 1.1rem;
}
.card h2, .panel h2 {
  margin: 0 0 .4rem;
  font-size: .8rem;
  font-weight: 600;
  letter-spacing: .06em;
  text-transform: uppercase;
  color: var(--muted);
}

.value { margin: 0; font-size: 1.45rem; font-weight: 650; }
.sub { margin: .3rem 0 0; color: var(--muted); font-size: .85rem; overflow-wrap: anywhere; }
.muted { color: var(--muted); font-size: .85rem; }

[data-level="ok"] { color: var(--ok); }
[data-level="warn"] { color: var(--warn); }
[data-level="bad"] { color: var(--bad); }

progress {
  width: 100%;
  height: .55rem;
  margin-top: .6rem;
  border: 0;
  border-radius: 99px;
  overflow: hidden;
  background: var(--panel-2);
  appearance: none;
}
progress::-webkit-progress-bar { background: var(--panel-2); }
progress::-webkit-progress-value { background: var(--accent); }
progress::-moz-progress-bar { background: var(--accent); }

.facts {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
  gap: .2rem 1.5rem;
  margin: .5rem 0 0;
}
.facts > div {
  display: flex;
  justify-content: space-between;
  gap: 1rem;
  padding: .45rem 0;
  border-bottom: 1px solid var(--border);
}
.facts dt { color: var(--muted); }
.facts dd { margin: 0; text-align: right; font-weight: 550; overflow-wrap: anywhere; }

.settings-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 1rem;
}
.kv { width: 100%; border-collapse: collapse; font-size: .9rem; }
.kv th, .kv td { padding: .35rem 0; border-bottom: 1px solid var(--border); text-align: left; }
.kv th { color: var(--muted); font-weight: 500; }
.kv td { text-align: right; overflow-wrap: anywhere; }

.notice {
  margin: 0;
  padding: .75rem 1rem;
  border: 1px solid var(--border);
  border-left: 3px solid var(--warn);
  border-radius: 8px;
  background: var(--panel);
  color: var(--muted);
}
code { background: var(--panel-2); padding: .1rem .35rem; border-radius: 5px; font-size: .85em; }

.placeholder { text-align: center; padding: 3rem 1rem; }

.footer {
  padding: 1.5rem;
  text-align: center;
  color: var(--muted);
  font-size: .8rem;
}

@media (max-width: 600px) {
  .topbar { flex-direction: column; align-items: flex-start; }
  .clock { text-align: left; }
  main { padding: .9rem; }
  .value { font-size: 1.25rem; }
}
EOF
```

```bash
cat > ~/surveillance/web/static/app.js <<'EOF'
"use strict";

// Fetches /api/status and fills in the elements present on the current page.
// Only textContent is ever set, so nothing from the server can be interpreted as HTML.
(() => {
  const refreshMs = Number(document.body.dataset.refresh) || 5000;
  let timer = null;

  function set(id, text, level) {
    const el = document.getElementById(id);
    if (!el) return;
    el.textContent = text;
    if (level) el.dataset.level = level; else delete el.dataset.level;
  }

  const gb = (bytes) => bytes == null ? "–" : (bytes / 1e9).toFixed(bytes >= 10e9 ? 1 : 2) + " GB";
  const localTime = (ms) => new Date(ms).toLocaleString([], { dateStyle: "medium", timeStyle: "medium" });

  function duration(seconds) {
    if (seconds == null) return "–";
    const d = Math.floor(seconds / 86400), h = Math.floor(seconds % 86400 / 3600), m = Math.floor(seconds % 3600 / 60);
    if (d) return `${d} d ${h} h`;
    if (h) return `${h} h ${m} min`;
    return m ? `${m} min` : `${Math.floor(seconds)} s`;
  }

  function ago(ms, now) {
    if (ms == null) return "never";
    const s = Math.max(0, (now - ms) / 1000);
    const rel = s < 60 ? "just now" : `${duration(s)} ago`;
    return `${localTime(ms)} (${rel})`;
  }

  function cameraState(rec) {
    if (!rec) return ["NOT RUNNING", "bad", "The recorder process is not running."];
    const phase = rec.recorder.phase;
    const table = {
      recording: ["ONLINE", "ok", ""],
      starting: ["STARTING", "warn", ""],
      retrying: ["OFFLINE", "bad", rec.recorder.message],
      stopping: ["STOPPING", "warn", ""],
      stopped: ["STOPPED", "bad", "The recorder was stopped."],
    };
    return table[phase] || [phase.toUpperCase(), "warn", ""];
  }

  function render(data) {
    const now = data.server_time;
    const rec = data.recorder;
    const sys = data.system;
    const db = data.database || {};

    const [camText, camLevel, camMessage] = cameraState(rec);
    set("camera-state", camText, camLevel);
    set("camera-message", camMessage || "");
    const dot = document.getElementById("header-dot");
    if (dot) dot.dataset.level = camLevel;

    const r = rec && rec.recording;
    if (!r) set("recording-state", "NO", "bad");
    else if (r.paused) set("recording-state", "PAUSED (storage full)", "bad");
    else if (r.write_error) set("recording-state", "ERROR", "bad");
    else set("recording-state", r.active ? "YES" : "NO", r.active ? "ok" : "bad");
    set("current-segment", r && r.current_segment ? `Writing ${r.current_segment}` : (r && r.write_error) || "");

    const m = rec && rec.motion;
    if (!m) set("motion-state", "–");
    else if (!m.enabled) set("motion-state", "DISABLED", "warn");
    else set("motion-state", m.active ? "DETECTED" : "NONE", m.active ? "warn" : "ok");
    set("last-motion", ago(db.last_motion_start ?? (m && m.last_event_start), now));

    const s = rec && rec.storage;
    if (s) {
      const level = { ok: "ok", low: "warn", critical: "bad", unavailable: "bad" }[s.state] || "warn";
      set("storage-summary", `${gb(s.used_bytes)} / ${gb(s.max_bytes)}`, level);
      set("storage-message", s.state === "ok" ? `${gb(s.free_bytes)} free on disk` : s.message, level === "ok" ? null : level);
      const bar = document.getElementById("storage-bar");
      if (bar) bar.value = Math.min(100, s.max_bytes ? s.used_bytes / s.max_bytes * 100 : 0);
      set("segment-count", String(s.segment_count));
      set("oldest", s.oldest_start ? localTime(s.oldest_start) : "none");
      set("newest", s.newest_start ? localTime(s.newest_start) : "none");
      set("disk-free", `${gb(s.free_bytes)} of ${gb(s.total_bytes)} (minimum kept free: ${gb(s.min_free_bytes)})`);
      if (db.bytes_per_second) {
        const usable = Math.min(s.max_bytes, s.used_bytes + s.free_bytes - s.min_free_bytes);
        set("retention", `about ${duration(usable / db.bytes_per_second)} of recordings`);
        set("bitrate", `${(db.bytes_per_second * 8 / 1e6).toFixed(2)} Mbit/s`);
      } else {
        set("retention", "not enough full segments yet");
        set("bitrate", "–");
      }
    } else {
      set("storage-summary", "–");
    }

    set("recordings-today", db.available ? String(db.recordings_today) : "–");
    set("events-today", db.available ? String(db.events_today) : "–");
    set("last-recording", ago(db.last_recording_end ?? (r && r.last_segment && r.last_segment.end_utc), now));
    set("recorder-uptime", rec && rec.recorder.phase !== "stopped" ? duration((now - rec.started_at) / 1000) : "–");
    if (rec) set("camera-config", `${rec.camera.resolution} @ ${rec.camera.fps} fps, ${(rec.camera.bitrate / 1e6).toFixed(1)} Mbit/s`);

    if (sys.cpu_percent == null) set("cpu", "measuring…");
    else set("cpu", `${sys.cpu_percent} % (load ${sys.load.join(" / ")})`, sys.cpu_percent < 70 ? "ok" : sys.cpu_percent < 90 ? "warn" : "bad");
    if (sys.memory) {
      const mem = sys.memory;
      set("memory", `${mem.used_percent} % (${gb(mem.used_bytes)} of ${gb(mem.total_bytes)})`, mem.used_percent < 80 ? "ok" : "warn");
    }
    const t = sys.temperature_c;
    set("temperature", t == null ? "–" : `${t} °C`, t == null ? null : t < 70 ? "ok" : t < 80 ? "warn" : "bad");
    const th = sys.throttled;
    if (!th) set("throttled", "unknown");
    else if (!th.flags.length) set("throttled", "OK (no under-voltage or throttling)", "ok");
    else set("throttled", th.flags.join(", "), th.flags.some((f) => f.endsWith("now")) ? "bad" : "warn");
    set("uptime", duration(sys.uptime_s));
    set("hostname", sys.hostname);

    const net = sys.network || {};
    if (net.wifi) {
      const dbm = net.wifi.signal_dbm;
      set("wifi", `${net.wifi.ssid || "connected"}, signal ${dbm} dBm`, dbm >= -67 ? "ok" : dbm >= -75 ? "warn" : "bad");
    } else if (net.ethernet === "up") {
      set("wifi", "not used (Ethernet connected)", "ok");
    } else {
      set("wifi", "not connected", "bad");
    }

    const clock = rec ? rec.clock_synced : null;
    set("clock-sync", clock === true ? "NTP-synchronised" : clock === false ? "NOT synchronised" : "unknown",
        clock === true ? "ok" : clock === false ? "bad" : "warn");
    if (!db.available) set("database", db.error ? `error: ${db.error}` : "not created yet", "warn");
    else set("database", rec && rec.database_healthy === false ? "write errors (see log)" : "OK",
             rec && rec.database_healthy === false ? "bad" : "ok");
    set("app-version", rec ? `recorder ${rec.version}, web ${data.web_version}` : `web ${data.web_version}`);
    set("updated-at", `Updated ${new Date(now).toLocaleTimeString()}`);
  }

  function showBanner(visible) {
    const banner = document.getElementById("connection-banner");
    if (banner) banner.hidden = !visible;
  }

  function schedule() {
    clearTimeout(timer);
    timer = setTimeout(refresh, refreshMs);
  }

  async function refresh() {
    if (document.visibilityState !== "visible") return;
    try {
      const response = await fetch("/api/status", { cache: "no-store", credentials: "same-origin" });
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      render(await response.json());
      showBanner(false);
    } catch (error) {
      showBanner(true);
    }
    schedule();
  }

  document.addEventListener("visibilitychange", () => {
    if (document.visibilityState === "visible") refresh();
    else clearTimeout(timer);
  });
  refresh();
})();
EOF
```

## Test procedure

You'll need **three windows**: two SSH sessions to the Pi and one terminal on your laptop.

**1. Pi, window 1: start the recorder.**

```bash
cd ~/surveillance && python3 -m app.main
```

**2. Pi, window 2: start the web interface.**

```bash
cd ~/surveillance && python3 -m app.web_main
```

Expect: `Web interface 0.8.0 listening on http://127.0.0.1:8080 (local only)`.

**3. Laptop: open the tunnel and keep this window open.**

```bash
ssh -N -L 8080:127.0.0.1:8080 ysak@ysak.local
```

Then browse to **http://localhost:8080** on the laptop.

**4. Check the pages:**
- **Dashboard:** Camera ONLINE, Recording YES, Motion NONE, and Storage `x GB / 45.0 GB`.
- **Walk in front of the camera.** Motion should switch to **DETECTED** within about 5–10 s, then back to NONE about 10 s after you stop moving.
- **System Status:** check CPU, RAM, temperature, `Power / throttling: OK`, your Wi-Fi network name and signal, and uptime.
- **Storage and Settings:** your real values should appear.

**5. Check the recorder states.**
- Press **Ctrl+C in window 1**. Within about 5 s the dashboard shows **STOPPED**, with a red dot.
- Start the recorder again. It goes to **STARTING** and then **ONLINE**.
- In a spare window run `pkill -9 -f "^python3 -m app.main"`. Within about 15 s the dashboard shows **NOT RUNNING**.
- Start the recorder again.

**6. Security checks, on the Pi:**

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8080/                                # 200
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: evil.example.com" http://127.0.0.1:8080/    # 400
ss -ltnp | grep 8080                                   # must show 127.0.0.1:8080, not 0.0.0.0
python3 tools/set_setting.py web.host 0.0.0.0          # must be refused
```

Then on the **laptop**, in a new window, check the web interface **isn't** reachable directly over your home network:

```bash
curl -m 5 http://ysak.local:8080/        # expected: "Connection refused" (or a timeout)
```

**7. Optional:** open the dashboard on your phone's browser through the laptop, or look at `logs/web.log`.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `Address already in use` | Another copy is running: `pkill -f "^python3 -m app.web_main"`, then start it again. |
| Tunnel says `channel 2: open failed` | The web interface isn't running in window 2. |
| `Bad request` in the browser | You used the Pi's IP or hostname. Use **http://localhost:8080** through the tunnel; the Host check blocks everything else by design. |
| Dashboard says NOT RUNNING while the recorder runs | Both programs must run as `ysak` from SSH sessions, so they share the same status folder. Send me the output of `echo $XDG_RUNTIME_DIR` from both windows. |
| `ModuleNotFoundError: flask` | `sudo apt install -y python3-flask python3-waitress` |

**Please send me:**
- a screenshot of the Dashboard and of System Status;
- the output of step 6;
- whether Motion switched to DETECTED when you walked past.

Phase 9 adds the Recordings browser (search, playback in the browser with seeking, download) and the Motion Events timeline, where clicking an event plays its recording.
