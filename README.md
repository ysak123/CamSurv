Phase 13 is ready as version 0.13.0. After you install it, the recorder and web interface run as systemd services under a dedicated, locked-down user, start at boot, and restart themselves after a crash or a hang. You won't need to start anything by hand again.

**First, one Phase 12 leftover.** Your ping went to 192.168.0.26 again, which is the laptop's home-network address. Please run it against the laptop's tailnet address, and tell me whether the policy saved with the new tests:

```bash
ping -c 3 100.85.224.6
```

Expected: `100% packet loss`.

## What Phase 13 changes

| What | Where |
|---|---|
| Program (root-owned, read-only to the services) | `/opt/surveillance` |
| Settings (the Settings page edits this file) | `/etc/surveillance/settings.json` |
| Recordings and database (moved, not copied) | `/var/lib/surveillance/` |
| Logs | `/var/log/surveillance/`, plus warnings in the system journal |
| Live status and live-view socket (in RAM) | `/run/surveillance/` |

- **Service user `cctv`:** no password and no login shell. It's in the `video` group for the camera, and nothing else.
- **Recorder service:** it can reach only camera devices and has no network access at all. It can't change its own settings or code. It reports its state to systemd (`Status: "recording"`).
- **Watchdog:** the recorder sends a keep-alive only while its supervisor loop is running. If that loop freezes, for example on a stuck camera driver, systemd kills and restarts it within about a minute.
- **Web service:** no devices, and recordings and database backups are read-only to it. A kernel rule allows only connections to and from the Pi itself (127.0.0.1), which is where `tailscale serve` and the SSH tunnel connect from.
- **Restart policy:** both services restart forever and never give up.
- **Journal size limit:** the system journal is capped at 200 MB so it can't fill the SD card.
- **New helper `sudo cctv-tool <tool>`:** runs the tools as `cctv`, for example `sudo cctv-tool manage_users list`.
- **Three compatibility fixes that surfaced while testing on Debian 13:**
  - `auth.py` now works with the argon2 library Debian ships (21.1). Your Pi probably has a newer copy installed only for your own user, which `cctv` can't see.
  - The password-hashing settings (64 MiB, 3 passes, 4 lanes) are now fixed in the code, so a different library version can't change them.
  - System Status reads throttling from sysfs, because the sandboxed web service can't run `vcgencmd`.

I tested this on a real Debian 13 system running systemd in a container, with the fake camera and a copy of the test recordings, database and admin account:

- **Fresh install:** the migration moved all recordings and the database, and both services started with 0 restarts.
- **Web tests:** the full browser test passed through the simulated Tailscale HTTPS proxy: login, live view, playback, download, and a settings save that the recorder picked up.
- **Restarts:** after `kill -9` the recorder was back in 5 s. After freezing it with SIGSTOP, systemd killed it on the watchdog after 51 s and it was recording again 6 s later.
- **Power cut:** I killed every process at once. Both services came up on boot, and the interrupted segment was recovered with 53.9 s playable.
- **Sandbox:**
  - The web service can't write recordings, backups or `/opt`, and can't read `/home`.
  - The recorder can't change its settings.
  - `systemd-analyze security` scores the services 1.3 and 1.4 (0 is fully locked down, 10 is unprotected).

Two things I couldn't test in the VM: real camera access under the device rules, and the kernel IP firewall (the container doesn't support it). Steps 4 and 7 cover them on the Pi.

## Step 1 — Stop the copies you started by hand

Press Ctrl+C in the recorder and web terminals, then check:

```bash
pgrep -af "app.main|app.web_main" || echo "nothing running"
```

## Step 2 — Apply the update

```bash
cd ~/surveillance
cat > update_to_0_13_0.py <<'PYEOF'
#!/usr/bin/env python3
"""Update the surveillance project from 0.12.0 to 0.13.0 (run from ~/surveillance)."""
import os, shutil, sys
from pathlib import Path

EDITS = [
    ('app/__init__.py',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.12.0"\n',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.13.0"\n'),
    ('app/auth.py',
     '\nfrom argon2 import PasswordHasher\nfrom argon2.exceptions import InvalidHashError, VerificationError, VerifyMismatchError\n\nlog = logging.getLogger("Auth")\n',
     '\nfrom argon2 import PasswordHasher\nfrom argon2.exceptions import InvalidHash, VerificationError, VerifyMismatchError\n\nlog = logging.getLogger("Auth")\n'),
    ('app/auth.py',
     'AUDIT_KEPT = 5000\nLOCKOUT_FORGET_S = 86400\n\nCOMMON_PASSWORDS = {\n',
     'AUDIT_KEPT = 5000\nLOCKOUT_FORGET_S = 86400\n# Argon2id, RFC 9106 "low memory" profile (64 MiB, 3 passes, 4 lanes). Fixed here rather than\n# taken from the library defaults, which differ between versions (Debian 13 ships 21.1).\nARGON2_PARAMS = {"time_cost": 3, "memory_cost": 65536, "parallelism": 4, "hash_len": 32, "salt_len": 16}\n\nCOMMON_PASSWORDS = {\n'),
    ('app/auth.py',
     '        self._max_age_s = session_hours * 3600\n        self._lock_threshold = lockout_threshold\n        self._hasher = PasswordHasher()\n        self._dummy_hash = self._hasher.hash(secrets.token_hex(16))\n        self._hash_slots = threading.BoundedSemaphore(2)\n',
     '        self._max_age_s = session_hours * 3600\n        self._lock_threshold = lockout_threshold\n        self._hasher = PasswordHasher(**ARGON2_PARAMS)\n        self._dummy_hash = self._hasher.hash(secrets.token_hex(16))\n        self._hash_slots = threading.BoundedSemaphore(2)\n'),
    ('app/auth.py',
     '            self._hasher.verify(password_hash, password)\n            return True\n        except (VerifyMismatchError, VerificationError, InvalidHashError):\n            return False\n        finally:\n',
     '            self._hasher.verify(password_hash, password)\n            return True\n        except (VerifyMismatchError, VerificationError, InvalidHash):\n            return False\n        finally:\n'),
    ('app/config.py',
     '\nPROJECT_ROOT = Path(__file__).resolve().parent.parent\nDEFAULT_CONFIG_PATH = PROJECT_ROOT / "config" / "settings.json"\n\nALLOWED_SEGMENT_SECONDS = (60, 120, 300, 600, 900, 1800, 3600)\n',
     '\nPROJECT_ROOT = Path(__file__).resolve().parent.parent\nSYSTEM_CONFIG_PATH = Path("/etc/surveillance/settings.json")\n\n\ndef _default_config_path() -> Path:\n    """$SURVEILLANCE_CONFIG, else the installed system file (Phase 13), else the project folder."""\n    override = os.environ.get("SURVEILLANCE_CONFIG")\n    if override:\n        return Path(override)\n    return SYSTEM_CONFIG_PATH if SYSTEM_CONFIG_PATH.parent.is_dir() else PROJECT_ROOT / "config" / "settings.json"\n\n\nDEFAULT_CONFIG_PATH = _default_config_path()\n\nALLOWED_SEGMENT_SECONDS = (60, 120, 300, 600, 900, 1800, 3600)\n'),
    ('app/logging_setup.py',
     'import json\nimport logging\nimport sys\nfrom datetime import datetime, timezone\n',
     'import json\nimport logging\nimport os\nimport sys\nfrom datetime import datetime, timezone\n'),
    ('app/logging_setup.py',
     '        file_handler.setFormatter(formatter)\n        root.addHandler(file_handler)\n\n    logging.captureWarnings(True)\n',
     '        file_handler.setFormatter(formatter)\n        root.addHandler(file_handler)\n        if settings.console and os.environ.get("JOURNAL_STREAM"):\n            # Under systemd the console goes to the journal; the file already has everything,\n            # so only problems are written twice (and show up in `systemctl status`).\n            console.setLevel(logging.WARNING)\n\n    logging.captureWarnings(True)\n'),
    ('app/main.py',
     'the pipeline is torn down and restarted with exponential backoff (5 s up to 60 s).\nSIGTERM/SIGINT (Ctrl+C) stop cleanly and finalise the current segment.\n"""\nfrom __future__ import annotations\n',
     'the pipeline is torn down and restarted with exponential backoff (5 s up to 60 s).\nSIGTERM/SIGINT (Ctrl+C) stop cleanly and finalise the current segment.\nUnder systemd (Phase 13) it reports readiness and status, and feeds the service watchdog.\n"""\nfrom __future__ import annotations\n'),
    ('app/main.py',
     '\nfrom app import __version__  # noqa: E402\nfrom app.camera import CameraError  # noqa: E402\nfrom app.clock import ClockMonitor  # noqa: E402\n',
     '\nfrom app import __version__  # noqa: E402\nfrom app import systemd_notify as sd  # noqa: E402\nfrom app.camera import CameraError  # noqa: E402\nfrom app.clock import ClockMonitor  # noqa: E402\n'),
    ('app/main.py',
     '    clock.start()\n    db = open_database(settings)\n    active: list[Recorder] = []\n    storage = StorageManager(settings, lambda: active[0].current_partial_path if active else None,\n',
     '    clock.start()\n    db = open_database(settings)\n    sd.heartbeat()\n    active: list[Recorder] = []\n    storage = StorageManager(settings, lambda: active[0].current_partial_path if active else None,\n'),
    ('app/main.py',
     '                             on_recovered=db.recording_recovered if db else None)\n    state = RecorderState()\n    publisher = StatusPublisher(settings, state, clock, storage, db, lambda: active[0] if active else None)\n    publisher.start()\n',
     '                             on_recovered=db.recording_recovered if db else None)\n    state = RecorderState()\n    state.add_listener(lambda: sd.status(f"{state.phase}: {state.message}" if state.message else state.phase))\n    publisher = StatusPublisher(settings, state, clock, storage, db, lambda: active[0] if active else None)\n    publisher.start()\n'),
    ('app/main.py',
     '    except Exception:  # noqa: BLE001 - the periodic check retries; recording matters more\n        log.exception("Initial storage check failed")\n    storage.start()\n    if db:\n        db.reconcile()\n    backoff = BACKOFF_INITIAL_S\n    try:\n',
     '    except Exception:  # noqa: BLE001 - the periodic check retries; recording matters more\n        log.exception("Initial storage check failed")\n    sd.heartbeat()\n    storage.start()\n    if db:\n        db.reconcile()\n    sd.heartbeat()\n    sd.ready()\n    backoff = BACKOFF_INITIAL_S\n    try:\n'),
    ('app/main.py',
     '                state.set("retrying", str(exc))\n                recorder.stop()\n                stop.wait(backoff)\n                backoff = min(backoff * 2, BACKOFF_MAX_S)\n                continue\n',
     '                state.set("retrying", str(exc))\n                recorder.stop()\n                sd.wait(stop, backoff)\n                backoff = min(backoff * 2, BACKOFF_MAX_S)\n                continue\n'),
    ('app/main.py',
     '            started = time.monotonic()\n            while not stop.wait(1.0):\n                problem = recorder.health()\n                if problem:\n',
     '            started = time.monotonic()\n            while not stop.wait(1.0):\n                sd.heartbeat()\n                problem = recorder.health()\n                if problem:\n'),
    ('app/main.py',
     '            recorder.stop()\n            if not stop.is_set():\n                stop.wait(backoff)\n                backoff = min(backoff * 2, BACKOFF_MAX_S)\n    finally:\n',
     '            recorder.stop()\n            if not stop.is_set():\n                sd.wait(stop, backoff)\n                backoff = min(backoff * 2, BACKOFF_MAX_S)\n    finally:\n'),
    ('app/main.py',
     '    stop = threading.Event()\n    terminate = threading.Event()\n\n    def request_stop(signum, _frame) -> None:\n        if not terminate.is_set():\n            log.info("Received %s, stopping", signal.Signals(signum).name)\n        terminate.set()\n        stop.set()\n',
     '    stop = threading.Event()\n    terminate = threading.Event()\n    watchdog = sd.Watchdog()\n    watchdog.start()\n\n    def request_stop(signum, _frame) -> None:\n        if not terminate.is_set():\n            log.info("Received %s, stopping", signal.Signals(signum).name)\n            sd.stopping()\n        terminate.set()\n        stop.set()\n'),
    ('app/main.py',
     '        log.exception("Fatal error")\n        return 1\n    log.info("Surveillance recorder stopped")\n    return 0\n',
     '        log.exception("Fatal error")\n        return 1\n    finally:\n        watchdog.stop()\n    log.info("Surveillance recorder stopped")\n    return 0\n'),
    ('app/system_info.py',
     '\nCACHE_S = 30.0\n\nTHROTTLE_FLAGS = {\n',
     '\nCACHE_S = 30.0\n# Same value as `vcgencmd get_throttled`, readable without access to the VideoCore device.\nTHROTTLED_SYSFS = ("/sys/devices/platform/soc/soc:firmware/get_throttled",)\n\nTHROTTLE_FLAGS = {\n'),
    ('app/system_info.py',
     '    @staticmethod\n    def throttled() -> dict | None:\n        out = _run(["vcgencmd", "get_throttled"])\n        if not out or "=" not in out:\n            return None\n        try:\n            value = int(out.split("=", 1)[1], 16)\n        except ValueError:\n            return None\n',
     '    @staticmethod\n    def throttled() -> dict | None:\n        raw = next((value for value in map(_read, THROTTLED_SYSFS) if value), None)\n        if raw is None:\n            out = _run(["vcgencmd", "get_throttled"])\n            if not out or "=" not in out:\n                return None\n            raw = out.split("=", 1)[1]\n        try:\n            value = int(raw, 16)\n        except ValueError:\n            return None\n'),
]

NEW_FILES = [
    ('app/systemd_notify.py', 0o644,
     '"""systemd service integration (Phase 13): readiness, status text and the watchdog.\n\nEverything here does nothing when the program is not started by systemd, so running it by\nhand in a terminal behaves exactly as before.\n\nWatchdog: the unit sets WatchdogSec=; systemd kills and restarts the recorder if it stops\nhearing from it. A background thread sends the keep-alive, but only while the supervisor\nloop in main.py keeps calling heartbeat(), so a hang anywhere in that loop (for example a\ncamera driver that never returns) leads to a restart instead of a silent stop.\n"""\nfrom __future__ import annotations\n\nimport logging\nimport os\nimport socket\nimport threading\nimport time\n\nlog = logging.getLogger("Systemd")\n\n_last_beat = time.monotonic()\n\n\ndef notify(message: str) -> bool:\n    address = os.environ.get("NOTIFY_SOCKET")\n    if not address:\n        return False\n    if address.startswith("@"):\n        address = "\\0" + address[1:]\n    try:\n        with socket.socket(socket.AF_UNIX, socket.SOCK_DGRAM | socket.SOCK_CLOEXEC) as sock:\n            sock.connect(address)\n            sock.sendall(message.encode("utf-8", "replace"))\n        return True\n    except OSError as exc:\n        log.debug("sd_notify failed: %s", exc)\n        return False\n\n\ndef ready() -> None:\n    notify("READY=1")\n\n\ndef stopping() -> None:\n    notify("STOPPING=1")\n\n\ndef status(text: str) -> None:\n    notify("STATUS=" + " ".join(text.split())[:200])\n\n\ndef heartbeat() -> None:\n    global _last_beat\n    _last_beat = time.monotonic()\n\n\ndef wait(stop: threading.Event, seconds: float) -> bool:\n    """Like stop.wait(seconds), but keeps the watchdog fed while waiting."""\n    deadline = time.monotonic() + seconds\n    while True:\n        heartbeat()\n        remaining = deadline - time.monotonic()\n        if remaining <= 0:\n            return stop.is_set()\n        if stop.wait(min(1.0, remaining)):\n            return True\n\n\nclass Watchdog:\n    def __init__(self) -> None:\n        usec = os.environ.get("WATCHDOG_USEC", "")\n        pid = os.environ.get("WATCHDOG_PID", "")\n        self.timeout_s = int(usec) / 1e6 if usec.isdigit() and (not pid or pid == str(os.getpid())) else 0.0\n        self._stop = threading.Event()\n        self._thread = threading.Thread(target=self._run, name="systemd-watchdog", daemon=True)\n        self._stalled = False\n\n    def start(self) -> None:\n        if self.timeout_s > 0:\n            heartbeat()\n            self._thread.start()\n            log.info("systemd watchdog on (%g s)", self.timeout_s)\n\n    def stop(self) -> None:\n        self._stop.set()\n\n    def _run(self) -> None:\n        interval, allowed = self.timeout_s / 3, self.timeout_s / 2\n        while not self._stop.wait(interval):\n            silent = time.monotonic() - _last_beat\n            if silent < allowed:\n                self._stalled = False\n                notify("WATCHDOG=1")\n            elif not self._stalled:\n                self._stalled = True\n                log.error("Supervisor loop silent for %.0f s; letting the systemd watchdog restart the recorder",\n                          silent)\n'),
    ('deploy/install.sh', 0o755,
     '#!/bin/bash\n# Install or update the surveillance services (Phase 13). Run from your project folder:\n#     sudo ~/surveillance/deploy/install.sh\n#\n# First run: creates the service user "cctv", copies the program to /opt/surveillance,\n# creates /etc/surveillance/settings.json from your current settings, moves the recordings\n# and database to /var/lib/surveillance, and starts both services at boot.\n# Later runs (after an update): copy the program again and restart the services. Settings,\n# recordings, database and accounts are kept.\nset -euo pipefail\n\nSRC="$(cd "$(dirname "$(readlink -f "$0")")/.." && pwd)"\nAPP=/opt/surveillance\nCONF_DIR=/etc/surveillance\nCONF="$CONF_DIR/settings.json"\nDATA=/var/lib/surveillance\nLOGS=/var/log/surveillance\nSVC_USER=cctv\nUNITS=(surveillance-recorder.service surveillance-web.service)\n\nsay() { printf \'\\n== %s\\n\' "$*"; }\ndie() { printf \'\\nERROR: %s\\n\' "$*" >&2; exit 1; }\n\n[ "$(id -u)" -eq 0 ] || die "run it with sudo: sudo $0"\n[ -d /run/systemd/system ] || die "systemd is not running on this machine"\n[ -f "$SRC/app/main.py" ] || die "$SRC does not look like the project folder"\n[ "$SRC" != "$APP" ] || die "run the copy in your project folder (~/surveillance/deploy/install.sh), not the one in $APP"\n\nsay "Checking for copies started by hand"\nfor pid in $(pgrep -f \'(^|/)python3 -m app\\.(main|web_main)( |$)\' || true); do\n    if [ "$(ps -o user= -p "$pid" | tr -d \' \')" != "$SVC_USER" ]; then\n        die "a recorder or web interface started by hand is still running (PID $pid). Press Ctrl+C in its terminal, then run this again."\n    fi\ndone\necho "none"\n\nsay "Stopping the services (if they are running)"\nsystemctl stop "${UNITS[@]}" 2>/dev/null || true\n\nsay "Service user $SVC_USER"\nif ! id "$SVC_USER" >/dev/null 2>&1; then\n    useradd --system --user-group --no-create-home --home-dir "$DATA" --shell /usr/sbin/nologin \\\n            --comment "Surveillance camera" "$SVC_USER"\n    echo "created"\nfi\nusermod -a -G video "$SVC_USER"\nADMIN="${SUDO_USER:-}"\nif [ -n "$ADMIN" ] && [ "$ADMIN" != root ] && ! id -nG "$ADMIN" | tr \' \' \'\\n\' | grep -qx "$SVC_USER"; then\n    usermod -a -G "$SVC_USER" "$ADMIN"\n    echo "$ADMIN added to group $SVC_USER (read access to recordings and logs; log in again to use it)"\nfi\nid "$SVC_USER"\n\nsay "Checking Python packages (as $SVC_USER)"\nrunuser -u "$SVC_USER" -- python3 - <<\'PY\' || die "install the missing packages listed above, then run this again"\nimport importlib, importlib.metadata as md, sys\nmodules = {"picamera2": "picamera2", "libcamera": None, "cv2": None, "numpy": "numpy", "av": "av",\n           "simplejpeg": "simplejpeg", "flask": "flask", "waitress": "waitress", "argon2": "argon2-cffi"}\nmissing, found = [], []\nfor name, dist in modules.items():\n    try:\n        module = importlib.import_module(name)\n    except Exception as exc:  # noqa: BLE001\n        missing.append(f"{name} ({exc})")\n        continue\n    try:\n        version = md.version(dist) if dist else getattr(module, "__version__", "")\n    except md.PackageNotFoundError:\n        version = getattr(module, "__version__", "")\n    found.append(f"{name} {version}".strip())\nprint("found: " + ", ".join(found))\nif missing:\n    print("missing: " + ", ".join(missing))\nsys.exit(1 if missing else 0)\nPY\n\nsay "Installing the program into $APP"\nrm -rf "$APP.new" "$APP.old"\nmkdir -p "$APP.new"\ntar -C "$SRC" --exclude=./recordings --exclude=./database --exclude=./logs --exclude=./config \\\n    --exclude=./.git --exclude=\'__pycache__\' --exclude=\'*.pyc\' --exclude=\'*.bak\' --exclude=\'./update_*.py\' \\\n    -cf - . | tar -C "$APP.new" -xf -\nchown -R root:root "$APP.new"\nchmod -R u=rwX,go=rX "$APP.new"\npython3 -m compileall -q -s "$APP.new" -p "$APP" "$APP.new/app" "$APP.new/tools" "$APP.new/deploy" >/dev/null\nif [ -d "$APP" ]; then mv "$APP" "$APP.old"; fi\nmv "$APP.new" "$APP"\nrm -rf "$APP.old"\npython3 -c "import sys; sys.path.insert(0, \'$APP\'); import app; print(\'version\', app.__version__)"\n\nsay "Folders"\ninstall -d -o "$SVC_USER" -g "$SVC_USER" -m 0750 "$CONF_DIR" "$DATA" "$DATA/recordings" "$DATA/database" \\\n    "$DATA/database/backups" "$LOGS"\necho "$CONF_DIR  $DATA  $LOGS"\n\nsay "Settings and data"\npython3 "$APP/deploy/setup_config.py" --project "$SRC" --config "$CONF" || \\\n    echo "WARNING: some items were not moved (see above); nothing was deleted"\nchown -R "$SVC_USER:$SVC_USER" "$CONF_DIR" "$DATA" "$LOGS"\nchmod -R u=rwX,g=rX,o= "$CONF_DIR" "$DATA" "$LOGS"\n\nsay "systemd units"\ninstall -m 0644 "$APP/deploy/surveillance-recorder.service" "$APP/deploy/surveillance-web.service" /etc/systemd/system/\ninstall -d -m 0755 /etc/systemd/journald.conf.d\ninstall -m 0644 "$APP/deploy/journald-surveillance.conf" /etc/systemd/journald.conf.d/50-surveillance.conf\ninstall -m 0755 "$APP/deploy/cctv-tool" /usr/local/sbin/cctv-tool\nsystemctl daemon-reload\nsystemctl try-restart systemd-journald.service || true\nsystemctl enable "${UNITS[@]}"\nsystemctl start "${UNITS[@]}"\n\nsay "Status (after 15 s)"\nsleep 15\nfailed=0\nfor unit in "${UNITS[@]}"; do\n    state="$(systemctl is-active "$unit" || true)"\n    restarts="$(systemctl show -p NRestarts --value "$unit")"\n    printf \'  %-32s %s, %s restart(s)\\n\' "$unit" "$state" "$restarts"\n    if [ "$state" != active ] || [ "$restarts" != 0 ]; then\n        failed=1\n        journalctl -u "$unit" -n 20 --no-pager -o cat | sed \'s/^/    | /\'\n    fi\ndone\nsystemctl show -p StatusText --value surveillance-recorder.service | sed \'s/^/  recorder: /\'\n[ "$failed" = 0 ] || die "a service is not running properly (log above). Fix it, then run this script again."\ncat <<EOF\n\nDone. Both services start at boot and restart themselves if they stop.\n  Status:        systemctl status surveillance-recorder surveillance-web\n  Problems:      journalctl -u surveillance-recorder -u surveillance-web -f\n  Full log:      sudo tail -f $LOGS/surveillance.log\n  Tools:         sudo cctv-tool set_setting --show     (sudo cctv-tool lists them all)\n  Deploy again:  sudo bash $SRC/deploy/install.sh     (after updating files in $SRC)\nEOF\n'),
    ('deploy/setup_config.py', 0o755,
     '#!/usr/bin/env python3\n"""Create the system settings file and move existing data into the service folders (Phase 13).\n\nCalled by deploy/install.sh as root; safe to run again. Only does work that is still needed:\n\n  * /etc/surveillance/settings.json missing: copy the settings from the project folder\n    (or defaults) with the paths changed to the system folders, validated like any save.\n  * Recordings / database still in the project folder: move them (a rename on the same\n    SD card: instant, nothing is copied). Existing files in the destination are never\n    overwritten; anything that cannot be moved is reported and left where it was.\n"""\nfrom __future__ import annotations\n\nimport argparse\nimport sys\nfrom dataclasses import asdict\nfrom pathlib import Path\n\nsys.path.insert(0, str(Path(__file__).resolve().parent.parent))\n\nfrom app.config import ConfigError, Settings, load_settings, save_settings, settings_from_dict  # noqa: E402\n\nSYSTEM_PATHS = {\n    "recordings_dir": "/var/lib/surveillance/recordings",\n    "database_dir": "/var/lib/surveillance/database",\n    "log_dir": "/var/log/surveillance",\n    "runtime_dir": "/run/surveillance",\n}\n\n\ndef system_settings(source: Settings | None) -> Settings:\n    data = asdict(source) if source else asdict(Settings())\n    for name, value in SYSTEM_PATHS.items():\n        current = data["paths"][name]\n        # A recordings folder on a USB drive (absolute, outside the project) stays where it is.\n        if name == "recordings_dir" and current and Path(current).is_absolute() and data["storage"]["required_mount"]:\n            continue\n        data["paths"][name] = value\n    return settings_from_dict(data)\n\n\ndef move_contents(old: Path, new: Path) -> tuple[int, list[str]]:\n    """Move every entry of `old` into `new` (rename). Returns (moved, problems)."""\n    moved, problems = 0, []\n    if not old.is_dir() or old.resolve() == new.resolve():\n        return 0, []\n    new.mkdir(parents=True, exist_ok=True)\n    for entry in sorted(old.iterdir()):\n        target = new / entry.name\n        if target.exists():\n            if entry.is_dir() and target.is_dir():\n                sub_moved, sub_problems = move_contents(entry, target)\n                moved += sub_moved\n                problems += sub_problems\n                try:\n                    entry.rmdir()\n                except OSError:\n                    pass\n            else:\n                problems.append(f"{entry} not moved: {target} already exists")\n            continue\n        try:\n            entry.rename(target)\n            moved += 1\n        except OSError as exc:\n            problems.append(f"{entry} not moved: {exc.strerror}")\n    return moved, problems\n\n\ndef main() -> int:\n    parser = argparse.ArgumentParser(description=__doc__.splitlines()[0])\n    parser.add_argument("--project", type=Path, required=True, help="the development copy, e.g. ~/surveillance")\n    parser.add_argument("--config", type=Path, required=True, help="system settings file to create")\n    args = parser.parse_args()\n\n    project_config = args.project / "config" / "settings.json"\n    old: Settings | None = None\n    if project_config.exists():\n        try:\n            old, _ = load_settings(project_config, create_if_missing=False)\n        except ConfigError as exc:\n            print(f"ERROR: {project_config} is invalid: {exc}")\n            return 2\n\n    if args.config.exists():\n        print(f"settings  : {args.config} already exists, kept as it is")\n        current, _ = load_settings(args.config, create_if_missing=False)\n    else:\n        current = system_settings(old)\n        save_settings(current, args.config)\n        print(f"settings  : created {args.config}" + (f" from {project_config}" if old else " with defaults"))\n\n    if old is None:\n        return 0\n    old_dirs = {"recordings": old.paths.recordings_dir, "database": old.paths.database_dir}\n    new_dirs = {"recordings": current.recordings_dir, "database": current.database_dir}\n    status = 0\n    project = args.project.resolve()\n    for name, old_value in old_dirs.items():\n        # Relative paths in the old settings were relative to the project folder.\n        source = (project / Path(old_value).expanduser()).resolve()\n        if not source.is_dir() or not source.is_relative_to(project):\n            continue\n        moved, problems = move_contents(source, new_dirs[name])\n        print(f"{name:10}: moved {moved} item(s) from {source} to {new_dirs[name]}")\n        for problem in problems:\n            print(f"  WARNING: {problem}")\n            status = 1\n    return status\n\n\nif __name__ == "__main__":\n    sys.exit(main())\n'),
    ('deploy/cctv-tool', 0o755,
     '#!/bin/sh\n# Run one of the surveillance tools as the service user "cctv", with the system settings.\n#   sudo cctv-tool set_setting --show\n#   sudo cctv-tool manage_users list\n#   sudo cctv-tool phase7_db_report\nset -eu\nif [ "$(id -u)" -ne 0 ]; then\n    echo "run it with sudo: sudo cctv-tool $*" >&2\n    exit 1\nfi\ntool="${1:-}"\ncase "$tool" in\n    "" | *[!a-z0-9_]*)\n        echo "usage: sudo cctv-tool TOOL [ARGUMENTS]   tools:" >&2\n        ls /opt/surveillance/tools | sed -n \'s/\\.py$//p\' | sed \'s/^/  /\' >&2\n        exit 2 ;;\nesac\nshift\nscript="/opt/surveillance/tools/$tool.py"\nif [ ! -f "$script" ]; then\n    echo "no tool called $tool (run sudo cctv-tool to list them)" >&2\n    exit 2\nfi\ncd /opt/surveillance\nexec runuser -u cctv -- env PYTHONPATH=/opt/surveillance PYTHONDONTWRITEBYTECODE=1 \\\n    SURVEILLANCE_CONFIG=/etc/surveillance/settings.json python3 "$script" "$@"\n'),
    ('deploy/surveillance-recorder.service', 0o644,
     '# Surveillance camera recorder (Phase 13). Installed by deploy/install.sh.\n# Runs as the unprivileged user "cctv": it can use the camera and write recordings, and\n# nothing else. It has no network access at all (the web interface talks to it through a\n# Unix socket in /run/surveillance).\n[Unit]\nDescription=Surveillance camera recorder\nAfter=local-fs.target\nRequiresMountsFor=/var/lib/surveillance\n# Never give up: keep restarting (every 5 s) however often it fails.\nStartLimitIntervalSec=0\n\n[Service]\nType=notify\nNotifyAccess=main\nUser=cctv\nGroup=cctv\nSupplementaryGroups=video\nWorkingDirectory=/opt/surveillance\nEnvironment=PYTHONPATH=/opt/surveillance SURVEILLANCE_CONFIG=/etc/surveillance/settings.json PYTHONDONTWRITEBYTECODE=1\nExecStart=/usr/bin/python3 -m app.main\nRestart=always\nRestartSec=5\n# The recorder feeds the watchdog only while its supervisor loop runs; if it hangs, systemd restarts it.\nWatchdogSec=60\nTimeoutStartSec=120\nTimeoutStopSec=30\n# Recording gets the CPU and SD card first when the Pi is busy.\nNice=-5\nIOSchedulingClass=best-effort\nIOSchedulingPriority=2\nMemoryMax=900M\n\nStateDirectory=surveillance\nStateDirectoryMode=0750\nLogsDirectory=surveillance\nLogsDirectoryMode=0750\nRuntimeDirectory=surveillance\nRuntimeDirectoryMode=0750\nRuntimeDirectoryPreserve=yes\nUMask=0027\n\n# Sandbox. Everything is read-only except the folders above; /etc/surveillance too (the recorder\n# only reads its settings). Only camera devices are reachable.\nProtectSystem=strict\nProtectHome=yes\nPrivateTmp=yes\nProtectProc=invisible\nDevicePolicy=closed\nDeviceAllow=char-video4linux rw\nDeviceAllow=char-media rw\nDeviceAllow=char-dma_heap rw\nIPAddressDeny=any\nRestrictAddressFamilies=AF_UNIX AF_NETLINK\nNoNewPrivileges=yes\nCapabilityBoundingSet=\nAmbientCapabilities=\nRestrictSUIDSGID=yes\nRestrictNamespaces=yes\nRestrictRealtime=yes\nLockPersonality=yes\nProtectKernelTunables=yes\nProtectKernelModules=yes\nProtectKernelLogs=yes\nProtectControlGroups=yes\nProtectClock=yes\nProtectHostname=yes\nSystemCallArchitectures=native\nSystemCallFilter=@system-service\nSystemCallErrorNumber=EPERM\n\n[Install]\nWantedBy=multi-user.target\n'),
    ('deploy/surveillance-web.service', 0o644,
     '# Surveillance web interface (Phase 13). Installed by deploy/install.sh.\n# Same unprivileged user as the recorder, but no camera access, the recordings are read-only\n# to it, and the kernel only lets it talk to this Pi itself (127.0.0.1): tailscale serve\n# and the SSH tunnel both connect from there.\n[Unit]\nDescription=Surveillance camera web interface\nAfter=network.target surveillance-recorder.service\nStartLimitIntervalSec=0\n\n[Service]\nType=simple\nUser=cctv\nGroup=cctv\nWorkingDirectory=/opt/surveillance\nEnvironment=PYTHONPATH=/opt/surveillance SURVEILLANCE_CONFIG=/etc/surveillance/settings.json PYTHONDONTWRITEBYTECODE=1\nExecStart=/usr/bin/python3 -m app.web_main\nRestart=always\nRestartSec=3\nTimeoutStopSec=15\nMemoryMax=400M\n\nStateDirectory=surveillance\nStateDirectoryMode=0750\nLogsDirectory=surveillance\nLogsDirectoryMode=0750\nRuntimeDirectory=surveillance\nRuntimeDirectoryMode=0750\nRuntimeDirectoryPreserve=yes\nUMask=0027\n\n# Sandbox. Writable: the settings folder (Settings page) and the database folder (logins and\n# sessions); the recordings and database backups are read-only.\nProtectSystem=strict\nReadWritePaths=/etc/surveillance\nReadOnlyPaths=-/var/lib/surveillance/recordings -/var/lib/surveillance/database/backups\nProtectHome=yes\nPrivateTmp=yes\nProtectProc=invisible\nPrivateDevices=yes\nIPAddressAllow=localhost\nIPAddressDeny=any\nRestrictAddressFamilies=AF_UNIX AF_INET AF_INET6 AF_NETLINK\nNoNewPrivileges=yes\nCapabilityBoundingSet=\nAmbientCapabilities=\nRestrictSUIDSGID=yes\nRestrictNamespaces=yes\nRestrictRealtime=yes\nLockPersonality=yes\nProtectKernelTunables=yes\nProtectKernelModules=yes\nProtectKernelLogs=yes\nProtectControlGroups=yes\nProtectClock=yes\nProtectHostname=yes\nSystemCallArchitectures=native\nSystemCallFilter=@system-service\nSystemCallErrorNumber=EPERM\n\n[Install]\nWantedBy=multi-user.target\n'),
    ('deploy/journald-surveillance.conf', 0o644,
     '# Installed as /etc/systemd/journald.conf.d/50-surveillance.conf by deploy/install.sh.\n# Caps the system journal so it can never fill the SD card; the recorder keeps its own\n# size-limited log files in /var/log/surveillance.\n[Journal]\nSystemMaxUse=200M\nSystemKeepFree=1G\nMaxRetentionSec=1month\n'),
]

texts = {}
for rel, old, new in EDITS:
    text = texts.setdefault(rel, Path(rel).read_text())
    if text.count(old) != 1:
        sys.exit(f"ABORTED, nothing changed: {rel} does not match the expected 0.12.0 code "
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
print("Update to 0.13.0 complete.")
PYEOF
python3 update_to_0_13_0.py
sha256sum app/__init__.py app/auth.py app/config.py app/logging_setup.py app/main.py app/system_info.py app/systemd_notify.py deploy/install.sh deploy/setup_config.py deploy/cctv-tool deploy/surveillance-recorder.service deploy/surveillance-web.service deploy/journald-surveillance.conf
```

Expected checksums:

```
5b101b8aca647f053a311dec1774618e77db16cbd55fd16cb90ac392eb9ad08e  app/__init__.py
8346f70bcd7e38c11dfcf82684a3118cda44a81c2adfbd072cf6496a0f7b5fb0  app/auth.py
005317889102077591a8c91b3b48af8169d7f36cca0a5c334b57e164176f4ebe  app/config.py
ad081540ad455f1a5e93a5983cff16a48222f37f419278fc59c08d30c389ab52  app/logging_setup.py
77699e69429d8855c45a9de72071f002aa26dda48de163cdfbaafb70b249011a  app/main.py
71393c63c87d15402d1f3d9becff543ea55b01ab5d73b2c0a09176141662564a  app/system_info.py
8abbbe56d8bba9363571dc537c4d1e9c1c98f7654e5b39609bbd95d540400fe5  app/systemd_notify.py
ac1d2f62578acfe7787fde809cb307bfa4553ef2b9fe407e9c54daa95afa2af2  deploy/install.sh
9772bf8523acbcf750bbe284e22ed96f38d2e9cb9e6cd4b71a23e638a3b452c0  deploy/setup_config.py
bf33ea14d2a89536b1b1e8a71a7b797717d362f6ac9e32800a52471a8a3f6632  deploy/cctv-tool
ec7bd0bc15390600fbdfb60378c65dd1aec7c3b94785ea09f9fa79f130d97e88  deploy/surveillance-recorder.service
3be52e3915f385cec45d7f8195af9b82b3344d940bc190f1392241d0b8d684d9  deploy/surveillance-web.service
6116b276eabf5b77905c5309376ee9e9ac30459250cad8fd2ebd88574c3a4f1c  deploy/journald-surveillance.conf
```

## Step 3 — Install the services

```bash
sudo bash ~/surveillance/deploy/install.sh
```

Expected: a series of `==` sections, ending with:

```
settings  : created /etc/surveillance/settings.json from /home/ysak/surveillance/config/settings.json
recordings: moved N item(s) from /home/ysak/surveillance/recordings to /var/lib/surveillance/recordings
database  : moved N item(s) from /home/ysak/surveillance/database to /var/lib/surveillance/database
...
  surveillance-recorder.service    active, 0 restart(s)
  surveillance-web.service         active, 0 restart(s)
  recorder: recording
```

Moving the data is instant, because it's a rename on the same SD card. If a problem comes up:

- **`missing: argon2 …` (or flask or waitress):** install the system copies with `sudo apt install python3-argon2 python3-flask python3-waitress` and run the installer again.
- **A service shows restarts:** the installer prints its log. Send it to me.

The installer also adds you to the `cctv` group. That takes effect after you log out and back in, and gives you read access to the logs and recordings.

## Step 4 — Check the camera and the website

```bash
systemctl status surveillance-recorder surveillance-web --no-pager
```

Both should show `active (running)`, and the recorder should show `Status: "recording"`. This is also the first real-camera test under the device restrictions.

Then open `https://cam01.tail1c1671.ts.net` on the laptop and check:

- You can log in with the same password.
- The Dashboard shows ONLINE.
- Recordings still lists your old recordings.
- Live View works, and old recordings play.

If the recorder isn't recording, send me the output of `journalctl -u surveillance-recorder -n 40 --no-pager`.

## Step 5 — Run the tools the new way

```bash
sudo cctv-tool phase4_check_segments
sudo cctv-tool manage_users list
sudo cctv-tool phase12_tailscale_check
```

Expected: `RESULT: SEGMENTS OK`, your admin account, and `TAILNET ACCESS OK`.

## Step 6 — Self-healing tests

**A. Crash.** The recorder should come back after about 5 s:

```bash
sudo systemctl kill -s KILL surveillance-recorder; sleep 10
systemctl show -p NRestarts -p StatusText surveillance-recorder
```

Expected: `NRestarts=1` and `StatusText=recording`.

**B. Hang (watchdog).** Freeze the recorder, then wait:

```bash
sudo systemctl kill -s STOP surveillance-recorder; sleep 75
journalctl -u surveillance-recorder --since "-3min" --no-pager | grep -E "Watchdog timeout|Started"
systemctl show -p NRestarts -p StatusText surveillance-recorder
```

Expected: `Watchdog timeout (limit 1min)!`, then `Started …`, then `NRestarts=2` and `StatusText=recording`.

**C. Reboot.** Run `sudo reboot`. After about a minute, open the website without touching the Pi; it should be ONLINE. Then connect over SSH and run:

```bash
journalctl -b -u surveillance-recorder -o short-monotonic --no-pager | grep -m1 "Started surveillance"
sudo grep -m1 "Started recording" /var/log/surveillance/surveillance.log
```

The number in brackets on the first line is how many seconds after power-on the recorder started.

## Step 7 — Sandbox checks

```bash
systemd-analyze security surveillance-recorder.service surveillance-web.service --no-pager | tail -3
sudo systemd-run --quiet --wait --pipe -p IPAddressAllow=localhost -p IPAddressDeny=any curl -m 5 -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8080/login
sudo systemd-run --quiet --wait --pipe -p IPAddressAllow=localhost -p IPAddressDeny=any curl -m 5 -s -o /dev/null -w "%{http_code}\n" https://1.1.1.1
cat /sys/devices/platform/soc/soc:firmware/get_throttled
```

Expected:
- **Security scores:** about 1.3 and 1.4, both marked `OK`.
- **First curl:** `200`, because the web's IP rule allows the Pi itself.
- **Second curl:** `000` (blocked). This proves the Pi's kernel enforces the web service's "local only" rule. If it prints `301`, the rule isn't enforced on your kernel; tell me.
- **`get_throttled`:** a number like `0`. System Status should now show the throttling field.

## Step 8 — Load

```bash
top -b -n 2 -d 5 -p "$(pgrep -d, -f 'app.main|app.web_main')"
```

Watch it once with Live View open.

**Your workflow from now on:**

- Don't start `python3 -m app.main` or `python3 -m app.web_main` by hand any more.
- `~/surveillance` stays your source copy. Its `recordings` and `database` folders are now empty, and its `config/settings.json` is no longer used. The live settings are in `/etc/surveillance`, and the Settings page edits them.
- For future updates: run the update script in `~/surveillance`, then `sudo bash ~/surveillance/deploy/install.sh`.
- The SSH tunnel still works.

Please send me:
1. The ping result and whether the policy saved.
2. The checksums.
3. The installer output.
4. The output of Steps 4 to 8.

Next is Phase 13b: Restart / Shut down buttons and a time-zone setting on the website, allowed through a narrow system permission rule (polkit) and protected by your password.
