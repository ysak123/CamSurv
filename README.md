Phase 11b, the editable Settings page, is ready for you to test on the Pi. It passed every check I ran here, including tampered form submissions and a hand-broken settings file. **Restart both the recorder and the web interface after updating**, because the recorder needs its new "watch the settings file" code.

## What Phase 11b adds

**Settings page**
- **Camera:** name, resolution (1296×972 full view, 1920×1080, 1280×720 or 640×480), frame rate, bitrate in Mbit/s, keyframe interval, rotate 180°, and **colour tuning**.
- **Recording:** segment length.
- **Motion detection:** on/off, sensitivity, minimum area, cooldown, trigger frames, analysis rate, how long to keep events.
- **Storage:** maximum GB, emergency free space, maximum age, retention.
- **Live view:** on/off, frame rate, quality, maximum viewers, auto-pause time.
- **Security:** idle logout, maximum session length, lockout threshold.
- **Interface:** dashboard refresh rate and log level.
- **Not editable in the browser:** the recordings and database folders, the web address and the required mount are shown read-only. A mistake there could lock you out or send recordings to the wrong disk, so they stay on the command line.

**Safety**
- Every submitted value is re-checked on the server by type and range, and drop-down values must be one of the allowed options; nothing sent by the browser is trusted. The complete result then goes through the same validation as the settings file, including cross-field rules such as "motion analysis rate can't exceed the camera frame rate".
- **Saving requires your password** unless you entered it in the last 5 minutes. Wrong passwords here count towards the same lockout as the login page, so a stolen session can't be used to guess your password.
- Every save is recorded in the security log with each old → new value.

**Changes apply automatically**
- **The recorder** notices a changed settings file within about 5 s and restarts its camera pipeline, so a few seconds of video are skipped. This also applies to `tools/set_setting.py`, so you no longer need to restart the recorder by hand.
- **Web-only changes** (security, dashboard refresh) don't interrupt recording at all.
- **A settings file broken by hand is ignored.** The recorder keeps running with the last good settings and logs why, and the Settings page shows a warning banner instead of saving.
- **The web interface restarts itself** after a save, and the page reloads with the new values. You stay logged in.

**Colour tuning for your NoIR camera.** The drop-down lists the tuning files for your sensor that libcamera ships (for example `ov5647.json` and `ov5647_noir.json`). Choosing `ov5647_noir.json` usually gives more natural daylight colours on cameras without an infrared filter.

## What I tested here (Chrome, fake camera, mock tuning folder)

- **Tuning choices:** only `ov5647.json` and `ov5647_noir.json` were offered. The mock folder also contained `imx219.json`, which was correctly left out because it belongs to a different sensor.
- **Invalid combination:** framerate 10 with analysis rate 15 was refused with "motion.analysis_fps cannot be higher than camera.framerate".
- **Tampered submissions**, sent by a script running inside the page, were all refused with `400`:
  - a 45-second segment length;
  - `../../etc/passwd` as the tuning file;
  - an unlisted resolution;
  - a `<script>` camera name.
- **No CSRF token:** refused with `403`, and the file was left untouched.
- **Password confirmation:**
  - It's required once the last confirmation is more than 5 minutes old. A wrong password was refused and logged as `reauth_failed`.
  - With the right password the change was saved. Recorder log: `Settings changed (camera.bitrate, camera.tuning_file, motion.sensitivity); restarting the recording pipeline`, then `Opened ov5647 … tuning ov5647_noir.json`.
  - The page reloaded by itself with the new values.
- **Web-only change:** saving the idle-timeout change gave `Settings changed (auth.idle_timeout_minutes); recording is not affected`.
- **Broken file:** I set a framerate of 500 by hand. The recorder logged it as invalid and kept its settings, and the page showed the warning and refused to save.
- **Update script:** turns your 0.11.0 files into files identical to the tested ones, and aborts safely if run twice.

## Update the files

Stop both programs first (Ctrl+C in both windows).

**1. Update script.** It changes 10 files and makes `.bak` backups:

```bash
cd ~/surveillance
cat > update_to_0_11_1.py <<'EOF'
#!/usr/bin/env python3
"""Update the surveillance project from 0.11.0 to 0.11.1 (run from ~/surveillance)."""
import shutil, sys
from pathlib import Path

EDITS = [
    ('app/__init__.py',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.11.0"\n',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.11.1"\n'),
    ('app/auth.py',
     '        with closing(self._connect()) as conn:\n            if not ok:\n                conn.execute("INSERT INTO login_failures (at) VALUES (?)", (now,))\n                conn.execute("DELETE FROM login_failures WHERE at <= ?", (now - GLOBAL_WINDOW_S,))\n                failed = (lock["failed"] if lock else 0) + 1\n                locked_until = 0\n                if failed >= self._lock_threshold:\n                    locked_until = now + min(LOCK_MAX_S, LOCK_BASE_S * 2 ** (failed - self._lock_threshold))\n                conn.execute("INSERT INTO lockouts (username, failed, locked_until, updated_at) VALUES (?, ?, ?, ?)"\n                             " ON CONFLICT(username) DO UPDATE SET failed = excluded.failed,"\n                             " locked_until = excluded.locked_until, updated_at = excluded.updated_at",\n                             (name, failed, locked_until, now))\n                conn.execute("DELETE FROM lockouts WHERE updated_at < ? AND locked_until < ?",\n                             (now - LOCKOUT_FORGET_S, now))\n                self.audit(conn, safe_label(name), "login_failed")\n                log.warning("Failed login for %r", safe_label(name))\n',
     '        with closing(self._connect()) as conn:\n            if not ok:\n                self._register_failure(conn, name, lock, now)\n                self.audit(conn, safe_label(name), "login_failed")\n                log.warning("Failed login for %r", safe_label(name))\n'),
    ('app/auth.py',
     '            log.info("User %s logged in", name)\n            return token, None\n\n    def _new_session(self, conn: sqlite3.Connection, user_id: int, user_agent: str) -> str:\n',
     '            log.info("User %s logged in", name)\n            return token, None\n\n    def _register_failure(self, conn: sqlite3.Connection, name: str, lock: sqlite3.Row | None, now: int) -> None:\n        conn.execute("INSERT INTO login_failures (at) VALUES (?)", (now,))\n        conn.execute("DELETE FROM login_failures WHERE at <= ?", (now - GLOBAL_WINDOW_S,))\n        failed = (lock["failed"] if lock else 0) + 1\n        locked_until = 0\n        if failed >= self._lock_threshold:\n            locked_until = now + min(LOCK_MAX_S, LOCK_BASE_S * 2 ** (failed - self._lock_threshold))\n        conn.execute("INSERT INTO lockouts (username, failed, locked_until, updated_at) VALUES (?, ?, ?, ?)"\n                     " ON CONFLICT(username) DO UPDATE SET failed = excluded.failed,"\n                     " locked_until = excluded.locked_until, updated_at = excluded.updated_at",\n                     (name, failed, locked_until, now))\n        conn.execute("DELETE FROM lockouts WHERE updated_at < ? AND locked_until < ?",\n                     (now - LOCKOUT_FORGET_S, now))\n\n    def _new_session(self, conn: sqlite3.Connection, user_id: int, user_agent: str) -> str:\n'),
    ('app/auth.py',
     '                                " WHERE user_id = ? ORDER BY last_seen DESC", (user_id,)).fetchall()\n\n    def change_password(self, session: dict, current: str, new: str) -> None:\n        with closing(self._connect()) as conn:\n',
     '                                " WHERE user_id = ? ORDER BY last_seen DESC", (user_id,)).fetchall()\n\n    def reauth_fresh(self, session: dict, max_age_s: int) -> bool:\n        return int(time.time()) - session["reauth_at"] <= max_age_s\n\n    def reauth(self, session: dict, password: str) -> None:\n        """Confirm the password for a sensitive action. Failures count towards the lockout,\n        so a stolen session cannot be used to guess the password."""\n        name, now = session["username"], int(time.time())\n        with closing(self._connect()) as conn:\n            lock = conn.execute("SELECT * FROM lockouts WHERE username = ?", (name,)).fetchone()\n            if lock and lock["locked_until"] > now:\n                raise AuthError("Too many wrong passwords. Try again in a few minutes.")\n            user = conn.execute("SELECT password_hash FROM users WHERE id = ?", (session["user_id"],)).fetchone()\n        ok = self._verify(user["password_hash"], password[:MAX_PASSWORD_LENGTH])\n        with closing(self._connect()) as conn:\n            if not ok:\n                self._register_failure(conn, name, lock, now)\n                self.audit(conn, name, "reauth_failed")\n                raise AuthError("That password is wrong.")\n            conn.execute("DELETE FROM lockouts WHERE username = ?", (name,))\n            conn.execute("UPDATE sessions SET reauth_at = ? WHERE token_hash = ?", (now, session["token_hash"]))\n        session["reauth_at"] = now\n\n    def change_password(self, session: dict, current: str, new: str) -> None:\n        with closing(self._connect()) as conn:\n'),
    ('app/camera.py',
     '        self.settings = settings\n        self.picam2 = None\n\n    def open(self) -> None:\n',
     '        self.settings = settings\n        self.picam2 = None\n        self.model: str | None = None\n\n    def open(self) -> None:\n'),
    ('app/camera.py',
     '        if s.camera_num >= len(cameras):\n            raise CameraError(f"camera {s.camera_num} not detected ({len(cameras)} camera(s) found)")\n        try:\n            self.picam2 = Picamera2(s.camera_num)\n        except (RuntimeError, IndexError) as exc:\n            reason = str(exc).rstrip(".")\n',
     '        if s.camera_num >= len(cameras):\n            raise CameraError(f"camera {s.camera_num} not detected ({len(cameras)} camera(s) found)")\n        tuning = None\n        if s.tuning_file:\n            try:\n                tuning = Picamera2.load_tuning_file(s.tuning_file)\n            except (RuntimeError, OSError, ValueError) as exc:\n                raise CameraError(f"cannot load tuning file {s.tuning_file}: {exc}") from exc\n        try:\n            self.picam2 = Picamera2(s.camera_num, tuning=tuning)\n        except (RuntimeError, IndexError) as exc:\n            reason = str(exc).rstrip(".")\n'),
    ('app/camera.py',
     '            raise CameraError(f"cannot configure camera: {exc}") from exc\n\n        model = self.picam2.camera_properties.get("Model", "unknown")\n        log.info("Opened %s: main %dx%d, lores %dx%d, %g fps, sensor mode %s", model, s.width, s.height,\n                 s.lores_width, s.lores_height, s.framerate,\n                 f"{sensor_size[0]}x{sensor_size[1]}" if sensor_size else "auto")\n\n    def _matching_sensor_size(self, size: tuple[int, int]) -> tuple[int, int] | None:\n',
     '            raise CameraError(f"cannot configure camera: {exc}") from exc\n\n        self.model = self.picam2.camera_properties.get("Model", "unknown")\n        log.info("Opened %s: main %dx%d, lores %dx%d, %g fps, sensor mode %s%s", self.model, s.width, s.height,\n                 s.lores_width, s.lores_height, s.framerate,\n                 f"{sensor_size[0]}x{sensor_size[1]}" if sensor_size else "auto",\n                 f", tuning {s.tuning_file}" if s.tuning_file else "")\n\n    def _matching_sensor_size(self, size: tuple[int, int]) -> tuple[int, int] | None:\n'),
    ('app/config.py',
     'MAX_ENCODER_PIXELS = 1920 * 1080\nCAMERA_NAME_PATTERN = re.compile(r"^[\\w][\\w .,\'()-]{0,39}$")\n\n\n',
     'MAX_ENCODER_PIXELS = 1920 * 1080\nCAMERA_NAME_PATTERN = re.compile(r"^[\\w][\\w .,\'()-]{0,39}$")\nTUNING_FILE_PATTERN = re.compile(r"^[a-z0-9_]{1,40}\\.json$")\n\n\n'),
    ('app/config.py',
     '    keyframe_seconds: float = 2.0\n    rotate180: bool = False\n\n    def validate(self) -> None:\n        _check_range("camera.camera_num", self.camera_num, 0, 3)\n        if not CAMERA_NAME_PATTERN.match(self.name):\n            raise ConfigError("camera.name must be 1-40 letters, digits, spaces or . , \' ( ) - _")\n',
     '    keyframe_seconds: float = 2.0\n    rotate180: bool = False\n    tuning_file: str = ""   # e.g. "ov5647_noir.json" for cameras without an infrared filter\n\n    def validate(self) -> None:\n        _check_range("camera.camera_num", self.camera_num, 0, 3)\n        if self.tuning_file and not TUNING_FILE_PATTERN.match(self.tuning_file):\n            raise ConfigError("camera.tuning_file must be empty or a file name like ov5647_noir.json")\n        if not CAMERA_NAME_PATTERN.match(self.name):\n            raise ConfigError("camera.name must be 1-40 letters, digits, spaces or . , \' ( ) - _")\n'),
    ('app/config.py',
     '\n\ndef save_settings(settings: Settings, path: Path = DEFAULT_CONFIG_PATH) -> None:\n    settings.validate()\n',
     '\n\ndef diff_settings(old: Settings, new: Settings) -> list[tuple[str, Any, Any]]:\n    """[("section.name", old_value, new_value), ...] for every value that differs."""\n    changes = []\n    old_data, new_data = asdict(old), asdict(new)\n    for section, values in new_data.items():\n        for name, value in values.items():\n            if old_data[section][name] != value:\n                changes.append((f"{section}.{name}", old_data[section][name], value))\n    return changes\n\n\ndef save_settings(settings: Settings, path: Path = DEFAULT_CONFIG_PATH) -> None:\n    settings.validate()\n'),
    ('app/main.py',
     'from app.camera import CameraError  # noqa: E402\nfrom app.clock import ClockMonitor  # noqa: E402\nfrom app.config import DEFAULT_CONFIG_PATH, ConfigError, Settings, load_settings  # noqa: E402\nfrom app.database import Database  # noqa: E402\nfrom app.logging_setup import setup_logging  # noqa: E402\n',
     'from app.camera import CameraError  # noqa: E402\nfrom app.clock import ClockMonitor  # noqa: E402\nfrom app.config import DEFAULT_CONFIG_PATH, ConfigError, Settings, diff_settings, load_settings  # noqa: E402\nfrom app.database import Database  # noqa: E402\nfrom app.logging_setup import setup_logging  # noqa: E402\n'),
    ('app/main.py',
     'BACKOFF_MAX_S = 60\nHEALTHY_RESET_S = 120\n\n\n',
     'BACKOFF_MAX_S = 60\nHEALTHY_RESET_S = 120\nSETTINGS_CHECK_S = 5.0\nRECORDER_SECTIONS = {"camera", "recording", "storage", "motion", "live", "paths", "logging"}\n\n\n'),
    ('app/main.py',
     '\n    stop = threading.Event()\n\n    def request_stop(signum, _frame) -> None:\n        if not stop.is_set():\n            log.info("Received %s, stopping", signal.Signals(signum).name)\n        stop.set()\n\n',
     '\n    stop = threading.Event()\n    terminate = threading.Event()\n\n    def request_stop(signum, _frame) -> None:\n        if not terminate.is_set():\n            log.info("Received %s, stopping", signal.Signals(signum).name)\n        terminate.set()\n        stop.set()\n\n'),
    ('app/main.py',
     '\n    try:\n        run(settings, stop)\n    except Exception:  # noqa: BLE001 - log it; systemd restarts the service\n        log.exception("Fatal error")\n',
     '\n    try:\n        while True:\n            watcher = SettingsWatcher(args.config, settings, stop)\n            watcher.start()\n            try:\n                run(settings, stop)\n            finally:\n                watcher.stop()\n            if terminate.is_set() or watcher.new_settings is None:\n                break\n            settings = watcher.new_settings\n            stop.clear()\n            setup_logging(settings.logging, settings.log_dir)\n            log.info("Recorder restarting with the new settings")\n    except Exception:  # noqa: BLE001 - log it; systemd restarts the service\n        log.exception("Fatal error")\n'),
    ('app/main.py',
     '\n\nif __name__ == "__main__":\n    sys.exit(main())\n',
     '\n\nclass SettingsWatcher:\n    """Restarts the recording pipeline when the settings file changes (e.g. saved from the web page).\n\n    An invalid file is logged and ignored. Changes that only affect the web interface\n    (web.*, auth.*) do not interrupt recording.\n    """\n\n    def __init__(self, path: Path, current: Settings, stop: threading.Event) -> None:\n        self._path = path\n        self._current = current\n        self._stop_recorder = stop\n        self._halt = threading.Event()\n        self._signature = self._file_signature()\n        self._thread = threading.Thread(target=self._run, name="settings-watcher", daemon=True)\n        self.new_settings: Settings | None = None\n\n    def _file_signature(self) -> tuple[int, int] | None:\n        try:\n            st = self._path.stat()\n        except OSError:\n            return None\n        return st.st_mtime_ns, st.st_size\n\n    def start(self) -> None:\n        self._thread.start()\n\n    def stop(self) -> None:\n        self._halt.set()\n        if self._thread.is_alive():\n            self._thread.join(timeout=10)\n\n    def _run(self) -> None:\n        while not self._halt.wait(SETTINGS_CHECK_S):\n            signature = self._file_signature()\n            if signature is None or signature == self._signature:\n                continue\n            self._signature = signature\n            try:\n                new, _ = load_settings(self._path, create_if_missing=False)\n            except ConfigError as exc:\n                log.error("The settings file changed but is invalid (%s); keeping the current settings", exc)\n                continue\n            changes = [key for key, _old, _new in diff_settings(self._current, new)]\n            recorder_changes = [key for key in changes if key.split(".")[0] in RECORDER_SECTIONS]\n            if not recorder_changes:\n                if changes:\n                    log.info("Settings changed (%s); recording is not affected", ", ".join(changes))\n                self._current = new\n                continue\n            log.info("Settings changed (%s); restarting the recording pipeline", ", ".join(recorder_changes))\n            self.new_settings = new\n            self._stop_recorder.set()\n            return\n\n\nif __name__ == "__main__":\n    sys.exit(main())\n'),
    ('app/recorder.py',
     '        return bool(self._output and self._output.paused)\n\n    def _boundary_loop(self, encoder: H264Encoder) -> None:\n        """Request a keyframe at each boundary so segments start on time, not up to 2 s late."""\n',
     '        return bool(self._output and self._output.paused)\n\n    @property\n    def camera_model(self) -> str | None:\n        return self._camera.model if self._camera else None\n\n    def _boundary_loop(self, encoder: H264Encoder) -> None:\n        """Request a keyframe at each boundary so segments start on time, not up to 2 s late."""\n'),
    ('app/status.py',
     '            "recorder": {"phase": self._state.phase, "message": self._state.message},\n            "camera": {"name": cam.name, "resolution": f"{cam.width}x{cam.height}", "fps": cam.framerate,\n                       "bitrate": cam.bitrate},\n            "recording": {\n                "active": recording and not recorder.paused,\n',
     '            "recorder": {"phase": self._state.phase, "message": self._state.message},\n            "camera": {"name": cam.name, "resolution": f"{cam.width}x{cam.height}", "fps": cam.framerate,\n                       "bitrate": cam.bitrate, "model": recorder.camera_model if recorder else None,\n                       "tuning_file": cam.tuning_file},\n            "recording": {\n                "active": recording and not recorder.paused,\n'),
    ('app/web.py',
     'import sqlite3\nimport time\nfrom dataclasses import asdict\nfrom datetime import datetime\n\nfrom flask import Flask, abort, jsonify, render_template, request\n\nfrom app import __version__\nfrom app.config import PROJECT_ROOT, Settings\nfrom app.database import DB_FILE_NAME, connect\nfrom app.status import read_status\n',
     'import sqlite3\nimport time\nfrom datetime import datetime\nfrom pathlib import Path\n\nfrom flask import Flask, abort, jsonify, render_template, request\n\nfrom app import __version__\nfrom app.config import DEFAULT_CONFIG_PATH, PROJECT_ROOT, Settings\nfrom app.database import DB_FILE_NAME, connect\nfrom app.status import read_status\n'),
    ('app/web.py',
     'from app.web_auth import install as install_auth\nfrom app.web_live import create_blueprint as create_live_blueprint\nfrom app.web_recordings import create_blueprint\n\n',
     'from app.web_auth import install as install_auth\nfrom app.web_live import create_blueprint as create_live_blueprint\nfrom app.web_settings import create_blueprint as create_settings_blueprint\nfrom app.web_recordings import create_blueprint\n\n'),
    ('app/web.py',
     '    ("rec.events", "Motion Events"),\n    ("storage", "Storage"),\n    ("settings_page", "Settings"),\n    ("system", "System Status"),\n]\n',
     '    ("rec.events", "Motion Events"),\n    ("storage", "Storage"),\n    ("cfg.settings_page", "Settings"),\n    ("system", "System Status"),\n]\n'),
    ('app/web.py',
     '\n\ndef create_app(settings: Settings) -> Flask:\n    app = Flask(__name__, template_folder=str(TEMPLATE_DIR), static_folder=str(STATIC_DIR))\n    app.config.update(JSON_SORT_KEYS=False, MAX_CONTENT_LENGTH=64 * 1024)\n',
     '\n\ndef create_app(settings: Settings, config_path: Path = DEFAULT_CONFIG_PATH) -> Flask:\n    app = Flask(__name__, template_folder=str(TEMPLATE_DIR), static_folder=str(STATIC_DIR))\n    app.config.update(JSON_SORT_KEYS=False, MAX_CONTENT_LENGTH=64 * 1024)\n'),
    ('app/web.py',
     '\n    # Registered after the Host check, so it runs second: login, CSRF and same-origin checks.\n    install_auth(app, settings)\n\n    @app.after_request\n',
     '\n    # Registered after the Host check, so it runs second: login, CSRF and same-origin checks.\n    store = install_auth(app, settings)\n    app.register_blueprint(create_settings_blueprint(settings, config_path, store))\n\n    @app.after_request\n'),
    ('app/web.py',
     '                               recordings_dir=settings.recordings_dir)\n\n    @app.get("/settings")\n    def settings_page():\n        return render_template("settings.html", title="Settings", page="settings_page", sections=asdict(settings))\n\n    @app.get("/api/status")\n    def api_status():\n',
     '                               recordings_dir=settings.recordings_dir)\n\n    @app.get("/api/status")\n    def api_status():\n'),
    ('app/web_main.py',
     '    host, port = settings.web.host, settings.web.port\n    log.info("Web interface %s listening on http://%s:%d (local only)", __version__, host, port)\n    serve(create_app(settings), host=host, port=port, threads=settings.live.max_viewers + SPARE_WORKER_THREADS,\n          ident="", clear_untrusted_proxy_headers=True)\n    return 0\n',
     '    host, port = settings.web.host, settings.web.port\n    log.info("Web interface %s listening on http://%s:%d (local only)", __version__, host, port)\n    serve(create_app(settings, args.config), host=host, port=port, threads=settings.live.max_viewers + SPARE_WORKER_THREADS,\n          ident="", clear_untrusted_proxy_headers=True)\n    return 0\n'),
    ('web/static/style.css',
     '}\nform.stack input:focus { outline: 2px solid var(--accent); outline-offset: 1px; }\n',
     '}\nform.stack input:focus { outline: 2px solid var(--accent); outline-offset: 1px; }\n\n/* ---- Phase 11b: settings form ---- */\n.settings-form { display: grid; gap: 1rem; }\n.field { display: grid; gap: .3rem; padding: .55rem 0; border-bottom: 1px solid var(--border); }\n.field > label { color: var(--text); font-size: .9rem; }\n.field input[type="text"], .field input[type="number"], .field input[type="password"], .field select {\n  padding: .45rem .55rem;\n  border: 1px solid var(--border);\n  border-radius: 8px;\n  background: var(--bg);\n  color: var(--text);\n  font: inherit;\n  width: 100%;\n}\n.field input:focus, .field select:focus { outline: 2px solid var(--accent); outline-offset: 1px; }\n.help { margin: 0; color: var(--muted); font-size: .8rem; }\n.error-box { border-left-color: var(--bad); color: var(--text); }\n.notice ul { margin: .4rem 0 0; padding-left: 1.2rem; }\n.save-bar { display: flex; flex-wrap: wrap; align-items: flex-end; gap: 1rem; position: sticky; bottom: 0; }\n.save-bar .field { border: 0; flex: 1; min-width: 240px; }\n'),
]

texts = {}
for rel, old, new in EDITS:
    text = texts.setdefault(rel, Path(rel).read_text())
    if text.count(old) != 1:
        sys.exit(f"ABORTED, nothing changed: {rel} does not match the expected 0.11.0 code "
                 f"(found {text.count(old)} matches for:\n{old})")
    texts[rel] = text.replace(old, new)
for rel, text in texts.items():
    shutil.copy2(rel, rel + ".bak")
    Path(rel).write_text(text)
    print(f"updated {rel}  (backup: {rel}.bak)")
print("Update to 0.11.1 complete.")
EOF
python3 update_to_0_11_1.py
```

It should print ten `updated …` lines and then `Update to 0.11.1 complete.`

**2. New file `app/web_settings.py`:**

```bash
cat > ~/surveillance/app/web_settings.py <<'EOF'
"""Editable settings page (Phase 11b).

* Every submitted value is parsed by its field type, choices are checked against the allowed list
  (never trusted from the browser), and the result goes through the same validation as the
  settings file, so the web page can never save something the recorder would reject.
* Saving requires the account password, unless it was confirmed in the last few minutes.
* The file is written atomically; the recorder notices the change and restarts its pipeline,
  and this web process restarts itself so it uses the new values too.
* Paths, network address and log file details are deliberately not editable from the browser.
"""
from __future__ import annotations

import logging
import os
import sys
import threading
from dataclasses import asdict, dataclass
from pathlib import Path

from flask import Blueprint, g, render_template, request

from app.auth import AuthError, AuthStore
from app.config import (ALLOWED_SEGMENT_SECONDS, LIVE_QUALITIES, LOG_LEVELS, PROJECT_ROOT, RETENTION_MODES,
                        ConfigError, Settings, diff_settings, load_settings, save_settings, settings_from_dict)
from app.status import read_status

log = logging.getLogger("Settings")

REAUTH_MAX_AGE_S = 300
RESTART_DELAY_S = 1.5
TUNING_DIRS = (Path("/usr/share/libcamera/ipa/rpi/vc4"), Path("/usr/local/share/libcamera/ipa/rpi/vc4"))
RESOLUTIONS = {
    "1296x972": (1296, 972, 640, 480, "1296 × 972 – full field of view (recommended for OV5647)"),
    "1920x1080": (1920, 1080, 640, 360, "1920 × 1080 – Full HD (a centre crop on OV5647)"),
    "1280x720": (1280, 720, 640, 360, "1280 × 720 – HD"),
    "640x480": (640, 480, 320, 240, "640 × 480 – smallest files"),
}
SEGMENT_LABELS = {60: "1 minute", 120: "2 minutes", 300: "5 minutes", 600: "10 minutes",
                  900: "15 minutes", 1800: "30 minutes", 3600: "1 hour"}


@dataclass(frozen=True)
class Field:
    key: str            # "section.name"
    label: str
    kind: str           # text | int | float | bool | select | mbps | resolution | tuning
    help: str = ""
    min: float | None = None
    max: float | None = None
    step: float | None = None
    choices: tuple = ()

    @property
    def section(self) -> str:
        return self.key.split(".")[0]

    @property
    def name(self) -> str:
        return self.key.split(".")[1]


GROUPS: list[tuple[str, list[Field]]] = [
    ("Camera", [
        Field("camera.name", "Camera name", "text", "Shown in the page header and in download file names."),
        Field("camera.resolution", "Recording resolution", "resolution"),
        Field("camera.framerate", "Frame rate (frames per second)", "float", "15 is plenty for surveillance.", 1, 30, 1),
        Field("camera.bitrate", "Bitrate (Mbit/s)", "mbps", "Higher is sharper but uses more storage. "
              "1 Mbit/s ≈ 0.45 GB per hour.", 0.25, 10, 0.25),
        Field("camera.keyframe_seconds", "Keyframe interval (seconds)", "float",
              "How often a full picture is stored. Smaller seeks faster, larger files slightly smaller.", 0.5, 10, 0.5),
        Field("camera.rotate180", "Rotate the image 180°", "bool"),
        Field("camera.tuning_file", "Colour tuning", "tuning",
              "Cameras without an infrared filter (NoIR) look pink in daylight; the _noir tuning corrects that."),
    ]),
    ("Recording", [
        Field("recording.segment_seconds", "Segment length", "select", "Recordings are split into files of this length.",
              choices=tuple((s, SEGMENT_LABELS[s]) for s in ALLOWED_SEGMENT_SECONDS)),
    ]),
    ("Motion detection", [
        Field("motion.enabled", "Motion detection on", "bool"),
        Field("motion.sensitivity", "Sensitivity (1-100)", "int", "Higher reacts to smaller changes in brightness.", 1, 100, 1),
        Field("motion.min_area_percent", "Minimum moving area (% of the picture)", "float",
              "Smaller movements are ignored. 0.5 % is roughly a 20 × 20 pixel patch.", 0.05, 50, 0.05),
        Field("motion.cooldown_seconds", "Cooldown (seconds)", "float",
              "An event ends only after this long without movement.", 1, 300, 1),
        Field("motion.trigger_frames", "Frames needed to start an event", "int",
              "Movement must be seen in this many analysed frames in a row.", 1, 20, 1),
        Field("motion.analysis_fps", "Analysed frames per second", "float", "", 1, 15, 1),
        Field("motion.keep_events_days", "Keep events without footage (days, 0 = forever)", "int", "", 0, 3650, 1),
    ]),
    ("Storage", [
        Field("storage.max_storage_gb", "Maximum recording storage (GB)", "float",
              "The oldest recordings are deleted to stay below this. Lowering it deletes footage immediately.",
              0.01, 100000, 0.5),
        Field("storage.min_free_gb", "Emergency minimum free space (GB)", "float",
              "Below half of this, recording pauses rather than filling the disk.", 0.1, 10000, 0.5),
        Field("storage.max_age_days", "Delete recordings older than (days, 0 = off)", "int", "", 0, 3650, 1),
        Field("storage.retention", "Retention", "select", "", choices=tuple((m, m.replace("_", " ")) for m in RETENTION_MODES)),
    ]),
    ("Live view", [
        Field("live.enabled", "Live view on", "bool"),
        Field("live.max_fps", "Maximum frames per second", "float", "", 1, 30, 1),
        Field("live.quality", "Picture quality", "select", "", choices=tuple((q, q) for q in LIVE_QUALITIES)),
        Field("live.max_viewers", "Maximum simultaneous viewers", "int", "", 1, 6, 1),
        Field("live.max_view_minutes", "Pause live view after (minutes)", "int", "", 1, 240, 1),
    ]),
    ("Security", [
        Field("auth.idle_timeout_minutes", "Log out after inactivity (minutes)", "int", "", 5, 1440, 1),
        Field("auth.session_max_hours", "Log out after at most (hours)", "int", "", 1, 168, 1),
        Field("auth.lockout_threshold", "Wrong passwords before a lockout", "int", "", 3, 20, 1),
    ]),
    ("Interface and logging", [
        Field("web.status_refresh_seconds", "Dashboard refresh (seconds)", "float", "", 2, 60, 1),
        Field("logging.level", "Log level", "select", "", choices=tuple((lv, lv) for lv in LOG_LEVELS)),
    ]),
]


def tuning_choices(model: str | None, current: str) -> list[tuple[str, str]]:
    names = set()
    for directory in TUNING_DIRS:
        if directory.is_dir():
            names.update(p.name for p in directory.glob("*.json") if model and p.stem.startswith(model))
    if current:
        names.add(current)
    return [("", "Default for this camera")] + [(n, n) for n in sorted(names)]


def resolution_choices(settings: Settings) -> list[tuple[str, str]]:
    current = f"{settings.camera.width}x{settings.camera.height}"
    choices = [(key, spec[4]) for key, spec in RESOLUTIONS.items()]
    if current not in RESOLUTIONS:
        choices.insert(0, ("custom", f"{current} (current, set in the settings file)"))
    return choices


def field_value(field: Field, data: dict):
    if field.kind == "resolution":
        return f"{data['camera']['width']}x{data['camera']['height']}"
    value = data[field.section][field.name]
    return round(value / 1e6, 2) if field.kind == "mbps" else value


def apply_form(data: dict, form, tuning: list[tuple[str, str]]) -> list[str]:
    """Write submitted values into `data` (a settings dict). Returns human-readable parse errors."""
    errors = []
    for _group, fields in GROUPS:
        for field in fields:
            raw = form.get(field.key)
            try:
                if field.kind == "bool":
                    value = field.key in form
                elif field.kind == "text":
                    value = (raw or "").strip()
                elif field.kind == "int":
                    value = int(raw)
                elif field.kind == "float":
                    value = float(raw)
                elif field.kind == "mbps":
                    value = int(round(float(raw) * 1_000_000))
                elif field.kind == "select":
                    allowed = {str(key): key for key, _label in field.choices}
                    value = allowed[raw]
                elif field.kind == "tuning":
                    if raw not in {key for key, _label in tuning}:
                        raise ValueError
                    value = raw
                elif field.kind == "resolution":
                    if raw != "custom":
                        width, height, lores_w, lores_h, _label = RESOLUTIONS[raw]
                        data["camera"].update(width=width, height=height, lores_width=lores_w, lores_height=lores_h)
                    continue
                else:
                    continue
            except (KeyError, TypeError, ValueError):
                errors.append(f"{field.label}: not a valid value.")
                continue
            data[field.section][field.name] = value
    return errors


def restart_web_process() -> None:
    """Replace this process with a fresh copy that reads the new settings (same PID, same arguments)."""
    log.info("Restarting the web interface to apply the new settings")
    logging.shutdown()
    env = dict(os.environ)
    env["PYTHONPATH"] = os.pathsep.join(p for p in (str(PROJECT_ROOT), env.get("PYTHONPATH")) if p)
    os.execve(sys.executable, [sys.executable, "-m", "app.web_main", *sys.argv[1:]], env)


def create_blueprint(settings: Settings, config_path: Path, store: AuthStore) -> Blueprint:
    bp = Blueprint("cfg", __name__)
    restart_lock = threading.Lock()

    def current_settings() -> tuple[Settings, str | None]:
        try:
            return load_settings(config_path, create_if_missing=False)[0], None
        except ConfigError as exc:
            return settings, f"The settings file is currently invalid ({exc}); showing the values in use."

    def render(data: dict, current: Settings, *, errors=(), message=None, changes=(), reloading=False, status=200):
        recorder = read_status(settings.runtime_dir) or {}
        model = (recorder.get("camera") or {}).get("model")
        tuning = tuning_choices(model, data["camera"]["tuning_file"])
        return render_template(
            "settings.html", title="Settings", page="cfg.settings_page", groups=GROUPS, data=data,
            value=lambda f: field_value(f, data), tuning=tuning, resolutions=resolution_choices(current),
            errors=list(errors), message=message, changes=list(changes), reloading=reloading,
            reauth_needed=not store.reauth_fresh(g.session, REAUTH_MAX_AGE_S),
            fixed={"Recordings folder": current.paths.recordings_dir, "Database folder": current.paths.database_dir,
                   "Web address": f"{current.web.host}:{current.web.port}",
                   "Required mount": current.storage.required_mount or "none"}), status

    @bp.get("/settings")
    def settings_page():
        current, problem = current_settings()
        return render(asdict(current), current, errors=[problem] if problem else [])

    @bp.post("/settings")
    def save():
        current, problem = current_settings()
        if problem:
            return render(asdict(current), current, errors=[problem], status=409)
        data = asdict(current)
        model = ((read_status(settings.runtime_dir) or {}).get("camera") or {}).get("model")
        errors = apply_form(data, request.form, tuning_choices(model, current.camera.tuning_file))
        if errors:
            return render(data, current, errors=errors, status=400)
        try:
            new = settings_from_dict(data)
        except ConfigError as exc:
            return render(data, current, errors=[str(exc)], status=400)
        changes = diff_settings(current, new)
        if not changes:
            return render(data, current, message="Nothing changed.")
        if not store.reauth_fresh(g.session, REAUTH_MAX_AGE_S):
            try:
                store.reauth(g.session, request.form.get("current_password", ""))
            except AuthError as exc:
                return render(data, current, errors=[str(exc)], status=403)

        save_settings(new, config_path)
        summary = "; ".join(f"{key}: {old} -> {value}" for key, old, value in changes)
        store.record(g.session["username"], "settings_changed", summary)
        log.info("Settings changed by %s: %s", g.session["username"], summary)
        if restart_lock.acquire(blocking=False):
            threading.Timer(RESTART_DELAY_S, restart_web_process).start()
        return render(asdict(new), new, message="Settings saved.", changes=changes, reloading=True)

    return bp
EOF
```

**3. Replace `web/templates/settings.html`, and add `web/static/settings.js`:**

```bash
cat > ~/surveillance/web/templates/settings.html <<'EOF'
{% extends "base.html" %}
{% block scripts %}<script src="{{ url_for('static', filename='settings.js') }}" defer></script>{% endblock %}
{% block content %}
{% if message %}
<div class="notice" {% if reloading %}id="reload-after" data-seconds="6"{% endif %}>
  <strong>{{ message }}</strong>
  {% if changes %}<ul>{% for key, old, new in changes %}<li><code>{{ key }}</code>: {{ old }} &rarr; {{ new }}</li>{% endfor %}</ul>{% endif %}
  {% if reloading %}<p>The web interface restarts now. The recorder applies recording changes within about
    10 seconds; a few seconds of video are skipped while the camera restarts. This page reloads by itself.</p>{% endif %}
</div>
{% endif %}
{% if errors %}
<div class="notice error-box" role="alert"><strong>Not saved:</strong>
  <ul>{% for e in errors %}<li>{{ e }}</li>{% endfor %}</ul>
</div>
{% endif %}

<form method="post" action="{{ url_for('cfg.save') }}" class="settings-form">
  <input type="hidden" name="csrf_token" value="{{ csrf_token }}">
  <div class="settings-grid">
  {% for group, fields in groups %}
    <section class="panel">
      <h2>{{ group }}</h2>
      {% for f in fields %}
      <div class="field">
        {% if f.kind == 'bool' %}
          <label class="check"><input type="checkbox" name="{{ f.key }}" {% if value(f) %}checked{% endif %}> {{ f.label }}</label>
        {% else %}
          <label for="{{ f.key }}">{{ f.label }}</label>
          {% if f.kind in ('select', 'tuning', 'resolution') %}
            {% set options = f.choices if f.kind == 'select' else (tuning if f.kind == 'tuning' else resolutions) %}
            <select id="{{ f.key }}" name="{{ f.key }}">
              {% for key, label in options %}
                <option value="{{ key }}" {% if key|string == value(f)|string %}selected{% endif %}>{{ label }}</option>
              {% endfor %}
            </select>
          {% elif f.kind == 'text' %}
            <input id="{{ f.key }}" type="text" name="{{ f.key }}" value="{{ value(f) }}" maxlength="40" required>
          {% else %}
            <input id="{{ f.key }}" type="number" name="{{ f.key }}" value="{{ value(f) }}"
                   {% if f.min is not none %}min="{{ f.min }}"{% endif %} {% if f.max is not none %}max="{{ f.max }}"{% endif %}
                   step="{{ '1' if f.kind == 'int' else 'any' }}" required>
          {% endif %}
        {% endif %}
        {% if f.help %}<p class="help">{{ f.help }}</p>{% endif %}
      </div>
      {% endfor %}
    </section>
  {% endfor %}
    <section class="panel">
      <h2>Not editable here</h2>
      <p class="help">These are changed only on the Pi with <code>tools/set_setting.py</code>, because a mistake
        could lock you out or send recordings to the wrong disk.</p>
      <table class="kv">
        {% for k, v in fixed.items() %}<tr><th scope="row">{{ k }}</th><td>{{ v }}</td></tr>{% endfor %}
      </table>
    </section>
  </div>

  <section class="panel save-bar">
    {% if reauth_needed %}
      <label class="field">Your password (required to save settings)
        <input type="password" name="current_password" autocomplete="current-password" maxlength="256" required>
      </label>
    {% else %}
      <p class="help">You confirmed your password in the last few minutes, so you can save without entering it again.</p>
    {% endif %}
    <button class="button primary" type="submit">Save settings</button>
  </section>
</form>
{% endblock %}
EOF

cat > ~/surveillance/web/static/settings.js <<'EOF'
"use strict";

// After saving, the web interface restarts; reload once it is back.
(() => {
  const notice = document.getElementById("reload-after");
  if (!notice) return;
  setTimeout(() => { window.location.href = "/settings"; }, Number(notice.dataset.seconds || 6) * 1000);
})();
EOF
```

## Test procedure

**1. Start both programs again:** `python3 -m app.main` in window 1 and `python3 -m app.web_main` in window 2. Log in and open **Settings**.

**2. Check which tuning files are offered:**

```bash
ls /usr/share/libcamera/ipa/rpi/vc4/ | grep ov5647
```

The **Colour tuning** drop-down should list the same files.

**3. Try the NoIR tuning** in daylight, if possible:
1. Choose **ov5647_noir.json** and click **Save settings**. It asks for your password if you logged in more than 5 minutes ago.
2. Within about 10 s, window 1 should show:
   ```text
   INFO Main: Settings changed (camera.tuning_file); restarting the recording pipeline
   INFO Camera: Opened ov5647: … tuning ov5647_noir.json
   ```
3. Compare the colours in **Live View**. If they're better, keep it; if not, set it back to **Default for this camera**.

**4. Checks:**
- **Invalid combination:** set Frame rate to **10** and Analysed frames per second to **15**, then save. It should refuse with "motion.analysis_fps cannot be higher than camera.framerate".
- **Web-only change:** change **Dashboard refresh** to 10 and save. Window 1 should log `… recording is not affected`, with no restart.
- **Security log:** run `python3 tools/manage_users.py log | tail -5`. It should show your `settings_changed` entries with old → new values.

**5. The command line applies automatically too.** Try:

```bash
python3 tools/set_setting.py motion.sensitivity 65
```

The recorder should log `Settings changed (motion.sensitivity); restarting …` within about 5 s, with no manual restart.

## Troubleshooting

| Symptom | Fix |
|---|---|
| The page doesn't come back after a save | Check window 2. The web interface restarts itself in place; if it stopped instead, start it again and send me the last lines of `logs/web.log`. |
| `cannot load tuning file …` in the recorder log | Pick another tuning file, or "Default", in Settings. The recorder keeps retrying and logs the error meanwhile. |
| "The settings file is currently invalid" | A hand edit broke the file. The message says which value; fix it with `tools/set_setting.py`. |

**Please send me:**
- whether saving worked and the recorder picked up the change;
- whether `ov5647_noir.json` improved the colours (a Live View screenshot with each setting helps);
- the security log lines.

Next is **Phase 12: Tailscale**. It gives you private, encrypted access from your phone and laptop with no router changes, a real HTTPS address with a valid certificate, and SSH only over the private network.
