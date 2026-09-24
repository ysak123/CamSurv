Phase 9 is ready for you to run on the Pi: the recordings browser, in-browser playback, downloads and the motion-event timeline all work in my tests here. Only the web interface changes, so the recorder can keep running while you update.

## What Phase 9 adds

**Recordings page**
- **A 24-hour timeline** of the chosen day. Blue shows what was recorded, yellow marks motion, and a red line marks "now". **Clicking a yellow marker plays that event.**
- **Day navigation:** previous/next day buttons, plus links to the last 14 days that have recordings.
- **Search** by date, a from/to time range, "motion only", or part of a segment name (e.g. `06-05`). A segment search without a date searches all days.
- **A list** showing each recording's time range (in the Pi's local time), length, size and flags (motion, recovered, clock not synced), each with **Play** and **Download** buttons.

**Motion Events page**
- The same timeline, and a list of events (newest first) showing time, duration and peak area.
- **Play opens the recording 5 seconds before the movement starts.**

**Player**
- Plays in the browser with normal seeking. The server supports byte-range requests, which Chrome needs to seek in these files.
- **Previous / Next segment** buttons, and a **"Play the next segment automatically"** option for watching a longer period in one go.
- **"Jump to 0:15"** buttons for each motion event in the recording.
- Details: length, size, status, clock sync, and file name.

**Download**
- The file is converted to a **standard MP4** without re-encoding: the playback index is moved to the start, which takes about a second. Recordings stay in the power-cut-safe format on disk, but downloads open in any player, including the VLC that showed a still image in Phase 3.
- The file gets a readable name, such as `Camera_1_2026-09-24_14-05-00.mp4`.
- Only one conversion runs at a time. If another is in progress, or the file is over 300 MB, you get the original file instead.

**Security**
- Recordings are looked up only by their database number, never by a path from the browser, so path-traversal tricks are impossible.
- Search inputs are strictly validated, and invalid values show an error message instead of running a query.
- Everything is still read-only and reachable only on the Pi itself. The Content-Security-Policy is strict enough that it even blocked my browser-automation tool until I disabled it for testing.

## What I tested here (Chrome, fake camera)

- **Every route:** the pages return 200, an unknown recording returns 404, and invalid dates, times and search text show friendly errors.
- **Byte ranges:** a range request returns `206 Partial Content` with the exact bytes.
- **Downloads:** the downloaded file has its index at the front (`ftyp, moov, mdat`), so it's a standard MP4 with no fragments.
- **A leak I found and fixed:** the first version left each converted download behind in `/tmp`, which is in RAM on your Pi. Now the file is deleted the moment it's opened, so nothing is ever left behind, even if a download is cancelled.
- **Playback, driven by Chrome:**
  - Clicking an event's Play opened `/recordings/7?t=15`, and the video started exactly at 15 s (the event 20 s in, minus the 5-second lead-in).
  - Seeking to 30 s landed on 30.
  - "Jump to" started playback at the right place.
  - With "play next automatically" ticked, the next segment started when the first ended.
  - Clicking a timeline marker opened the right recording.
- **A timeline bug I found and fixed:** a name clash in the template made the timeline draw empty.
- **Phone layout:** the table overflowed the screen. It now fits a 390-pixel phone screen, with the Size column hidden.

## Update the files

Run everything from `~/surveillance`.

**1. Version number:**

```bash
cd ~/surveillance
sed -i 's/__version__ = "0.8.0"/__version__ = "0.9.0"/' app/__init__.py && cat app/__init__.py
```

**2. Replace `app/web.py`:**

```bash
cat > ~/surveillance/app/web.py <<'EOF'
"""Web interface (Flask): dashboard, system status, storage and settings pages.

Recordings, playback and the motion timeline live in app/web_recordings.py.

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
from app.web_recordings import create_blueprint

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
    ("rec.recordings", "Recordings"),
    ("rec.events", "Motion Events"),
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
    app.register_blueprint(create_blueprint(settings))
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

**3. New `app/web_recordings.py`:**

```bash
cat > ~/surveillance/app/web_recordings.py <<'EOF'
"""Recordings browser, playback, download and the motion-event timeline (Phase 9).

Files are always looked up by their database id and then resolved inside recordings_dir;
a path from the browser is never used, so path traversal is impossible. Video is served
with HTTP Range support, which browsers need to seek in fragmented MP4 files.
"""
from __future__ import annotations

import os
import re
import shutil
import sqlite3
import subprocess
import tempfile
import threading
from contextlib import closing
from dataclasses import dataclass
from datetime import date, datetime, time as dtime, timedelta
from pathlib import Path

from flask import Blueprint, abort, render_template, request, send_file, url_for

from app.config import Settings
from app.database import DB_FILE_NAME, connect

DATE_RE = re.compile(r"^\d{4}-\d{2}-\d{2}$")
TIME_RE = re.compile(r"^([01]\d|2[0-3]):[0-5]\d$")
SEGMENT_RE = re.compile(r"^[0-9A-Za-z:._/-]{1,40}$")
PRE_ROLL_S = 5
MAX_ROWS = 1000
DAY_MINUTES = 1440
EVENT_MIN_WIDTH_MIN = 4
REMUX_MAX_BYTES = 300_000_000
REMUX_TIMEOUT_S = 120
RECENT_DAYS = 14


class FilterError(ValueError):
    pass


@dataclass(frozen=True)
class Filters:
    day: date
    day_given: bool
    time_from: str
    time_to: str
    motion_only: bool
    segment: str


def today() -> date:
    return datetime.now().astimezone().date()


def parse_filters(args) -> Filters:
    raw_day = args.get("date", "").strip()
    day, day_given = today(), False
    if raw_day:
        if not DATE_RE.match(raw_day):
            raise FilterError("Date must look like 2026-09-24.")
        try:
            day, day_given = date.fromisoformat(raw_day), True
        except ValueError:
            raise FilterError("That date does not exist.") from None
    time_from, time_to = args.get("from", "").strip(), args.get("to", "").strip()
    for value in (time_from, time_to):
        if value and not TIME_RE.match(value):
            raise FilterError("Times must look like 08:30 (24-hour clock).")
    segment = args.get("segment", "").strip()
    if segment and not SEGMENT_RE.match(segment):
        raise FilterError("Segment search may only contain letters, digits and : . _ / -")
    return Filters(day, day_given, time_from, time_to, args.get("motion") == "1", segment)


def day_bounds(day: date) -> tuple[int, int]:
    start = datetime.combine(day, dtime.min).astimezone()
    end = datetime.combine(day + timedelta(days=1), dtime.min).astimezone()
    return int(start.timestamp() * 1000), int(end.timestamp() * 1000)


def local(ms: int | None) -> datetime | None:
    return datetime.fromtimestamp(ms / 1000) if ms is not None else None


def minutes(hhmm: str) -> int:
    hours, mins = hhmm.split(":")
    return int(hours) * 60 + int(mins)


def fmt_duration(ms: int | None) -> str:
    if ms is None:
        return "?"
    seconds = round(ms / 1000)
    return f"{seconds // 60} min {seconds % 60:02d} s" if seconds >= 60 else f"{seconds} s"


def escape_like(text: str) -> str:
    return text.replace("\\", "\\\\").replace("%", "\\%").replace("_", "\\_")


def safe_filename(text: str) -> str:
    return re.sub(r"[^A-Za-z0-9_-]+", "_", text).strip("_") or "camera"


def present(row: sqlite3.Row) -> dict:
    return {
        "id": row["id"],
        "start": local(row["start_utc"]),
        "end": local(row["end_utc"]),
        "duration": fmt_duration(row["duration_ms"]),
        "size_mb": row["size_bytes"] / 1e6,
        "has_motion": bool(row["has_motion"]),
        "recovered": row["status"] == "recovered",
        "clock_unsynced": row["clock_synced"] == 0,
        "rel_path": row["rel_path"],
    }


def create_blueprint(settings: Settings) -> Blueprint:
    bp = Blueprint("rec", __name__)
    root = settings.recordings_dir.resolve()
    db_path = settings.database_dir / DB_FILE_NAME
    remux_slot = threading.BoundedSemaphore(1)

    def open_db() -> sqlite3.Connection:
        if not db_path.exists():
            abort(503)
        return connect(db_path, read_only=True)

    def recording_row(conn: sqlite3.Connection, rid: int) -> sqlite3.Row:
        row = conn.execute("SELECT * FROM recordings WHERE id = ?", (rid,)).fetchone()
        if row is None:
            abort(404)
        return row

    def file_path(row: sqlite3.Row) -> Path:
        path = (root / row["rel_path"]).resolve()
        if not path.is_relative_to(root) or not path.is_file():
            abort(404)
        return path

    def event_link(conn: sqlite3.Connection, event: sqlite3.Row) -> tuple[str | None, list[dict]]:
        linked = conn.execute(
            "SELECT r.id, r.start_utc, r.rel_path FROM motion_event_recordings l JOIN recordings r"
            " ON r.id = l.recording_id WHERE l.event_id = ? ORDER BY r.start_utc", (event["id"],)).fetchall()
        if not linked:
            return None, []
        first = linked[0]
        offset = max(0, (event["start_utc"] - first["start_utc"]) / 1000 - PRE_ROLL_S)
        url = url_for("rec.recording", rid=first["id"], t=f"{offset:.0f}")
        return url, [{"id": r["id"], "rel_path": r["rel_path"]} for r in linked]

    def timeline(conn: sqlite3.Connection, start_ms: int, end_ms: int) -> dict:
        spans: list[list[float]] = []
        for row in conn.execute("SELECT start_utc, end_utc FROM recordings WHERE start_utc < ? AND end_utc > ?"
                                " ORDER BY start_utc", (end_ms, start_ms)):
            a = max(0.0, (row["start_utc"] - start_ms) / 60000)
            b = min(float(DAY_MINUTES), (row["end_utc"] - start_ms) / 60000)
            if spans and a - spans[-1][1] < 0.5:
                spans[-1][1] = max(spans[-1][1], b)
            else:
                spans.append([a, b])
        events = []
        for event in conn.execute("SELECT * FROM motion_events WHERE start_utc >= ? AND start_utc < ?"
                                  " ORDER BY start_utc", (start_ms, end_ms)):
            url, _ = event_link(conn, event)
            x = (event["start_utc"] - start_ms) / 60000
            width = max(EVENT_MIN_WIDTH_MIN, (event["duration_ms"] or 0) / 60000)
            events.append({"x": round(x, 2), "w": round(width, 2), "url": url,
                           "label": f"{local(event['start_utc']):%H:%M:%S} motion, {fmt_duration(event['duration_ms'])}"})
        now_ms = int(datetime.now().timestamp() * 1000)
        now_x = (now_ms - start_ms) / 60000 if start_ms <= now_ms < end_ms else None
        return {"spans": [{"x": round(a, 2), "w": round(max(b - a, 0.5), 2)} for a, b in spans],
                "events": events, "now_x": now_x}

    def recent_days(conn: sqlite3.Connection) -> list[str]:
        rows = conn.execute("SELECT DISTINCT date(start_utc / 1000, 'unixepoch', 'localtime') AS d FROM recordings"
                            " ORDER BY d DESC LIMIT ?", (RECENT_DAYS,)).fetchall()
        return [row["d"] for row in rows]

    @bp.get("/recordings")
    def recordings():
        error = None
        try:
            filters = parse_filters(request.args)
        except FilterError as exc:
            error = str(exc)
            filters = parse_filters({})
        start_ms, end_ms = day_bounds(filters.day)
        sql = "SELECT * FROM recordings WHERE 1 = 1"
        params: list = []
        if filters.day_given or not filters.segment:
            q_start = start_ms + minutes(filters.time_from) * 60000 if filters.time_from else start_ms
            q_end = start_ms + (minutes(filters.time_to) + 1) * 60000 if filters.time_to else end_ms
            sql += " AND start_utc < ? AND COALESCE(end_utc, start_utc) > ?"
            params += [q_end, q_start]
        if filters.motion_only:
            sql += " AND has_motion = 1"
        if filters.segment:
            sql += " AND rel_path LIKE ? ESCAPE '\\'"
            params.append(f"%{escape_like(filters.segment)}%")
        sql += " ORDER BY start_utc LIMIT ?"
        params.append(MAX_ROWS)
        with closing(open_db()) as conn:
            items = [present(row) for row in conn.execute(sql, params)]
            tl = timeline(conn, start_ms, end_ms)
            days = recent_days(conn)
        return render_template("recordings.html", title="Recordings", page="rec.recordings", filters=filters,
                               items=items, day_timeline=tl, days=days, error=error, max_rows=MAX_ROWS,
                               prev_day=filters.day - timedelta(days=1), next_day=filters.day + timedelta(days=1),
                               all_days=bool(filters.segment and not filters.day_given))

    @bp.get("/events")
    def events():
        error = None
        try:
            filters = parse_filters({"date": request.args.get("date", "")})
        except FilterError as exc:
            error = str(exc)
            filters = parse_filters({})
        start_ms, end_ms = day_bounds(filters.day)
        items = []
        with closing(open_db()) as conn:
            for event in conn.execute("SELECT * FROM motion_events WHERE start_utc >= ? AND start_utc < ?"
                                      " ORDER BY start_utc DESC", (start_ms, end_ms)).fetchall():
                url, files = event_link(conn, event)
                items.append({"start": local(event["start_utc"]), "duration": fmt_duration(event["duration_ms"]),
                              "ongoing": event["end_utc"] is None, "peak": event["peak_area_pct"] or 0,
                              "url": url, "files": files})
            tl = timeline(conn, start_ms, end_ms)
            days = [row["d"] for row in conn.execute(
                "SELECT DISTINCT date(start_utc / 1000, 'unixepoch', 'localtime') AS d FROM motion_events"
                " ORDER BY d DESC LIMIT ?", (RECENT_DAYS,))]
        return render_template("events.html", title="Motion Events", page="rec.events", day=filters.day,
                               items=items, day_timeline=tl, days=days, error=error,
                               prev_day=filters.day - timedelta(days=1), next_day=filters.day + timedelta(days=1))

    @bp.get("/recordings/<int:rid>")
    def recording(rid: int):
        try:
            start_at = min(max(float(request.args.get("t", "0")), 0.0), 86400.0)
        except ValueError:
            start_at = 0.0
        with closing(open_db()) as conn:
            row = recording_row(conn, rid)
            previous = conn.execute("SELECT id FROM recordings WHERE start_utc < ? ORDER BY start_utc DESC LIMIT 1",
                                    (row["start_utc"],)).fetchone()
            following = conn.execute("SELECT id FROM recordings WHERE start_utc > ? ORDER BY start_utc LIMIT 1",
                                     (row["start_utc"],)).fetchone()
            events = [{"start": local(e["start_utc"]), "duration": fmt_duration(e["duration_ms"]),
                       "offset": max(0, round((e["start_utc"] - row["start_utc"]) / 1000 - PRE_ROLL_S))}
                      for e in conn.execute(
                          "SELECT e.* FROM motion_event_recordings l JOIN motion_events e ON e.id = l.event_id"
                          " WHERE l.recording_id = ? ORDER BY e.start_utc", (rid,))]
        return render_template("player.html", title="Recording", page="rec.recordings", rec=present(row),
                               events=events, start_at=start_at,
                               prev_url=url_for("rec.recording", rid=previous["id"]) if previous else None,
                               next_url=url_for("rec.recording", rid=following["id"]) if following else None)

    @bp.get("/recordings/<int:rid>/video")
    def video(rid: int):
        with closing(open_db()) as conn:
            path = file_path(recording_row(conn, rid))
        return send_file(path, mimetype="video/mp4", conditional=True, etag=True, max_age=0)

    @bp.get("/recordings/<int:rid>/download")
    def download(rid: int):
        with closing(open_db()) as conn:
            row = recording_row(conn, rid)
        path = file_path(row)
        name = f"{safe_filename(settings.camera.name)}_{local(row['start_utc']):%Y-%m-%d_%H-%M-%S}.mp4"
        # Recordings are fragmented MP4 (safe against power cuts). Downloads are converted to a
        # standard MP4 without re-encoding, so every media player handles them.
        if path.stat().st_size <= REMUX_MAX_BYTES and shutil.which("ffmpeg") and remux_slot.acquire(blocking=False):
            try:
                converted = remux(path)
            finally:
                remux_slot.release()
            if converted is not None:
                # Unlink straight away: the open handle keeps the data readable until the transfer
                # ends, and nothing is left behind even if the client disconnects or we crash.
                handle = open(converted, "rb")
                converted.unlink()
                return send_file(handle, mimetype="video/mp4", as_attachment=True, download_name=name)
        return send_file(path, mimetype="video/mp4", as_attachment=True, download_name=name, conditional=True)

    return bp


def remux(source: Path) -> Path | None:
    fd, tmp_name = tempfile.mkstemp(prefix="surveillance-download-", suffix=".mp4")
    os.close(fd)
    tmp = Path(tmp_name)
    try:
        result = subprocess.run(["ffmpeg", "-v", "error", "-y", "-i", str(source), "-map", "0", "-c", "copy",
                                 "-movflags", "+faststart", "-f", "mp4", str(tmp)],
                                capture_output=True, timeout=REMUX_TIMEOUT_S, check=False)
    except (OSError, subprocess.TimeoutExpired):
        tmp.unlink(missing_ok=True)
        return None
    if result.returncode != 0 or tmp.stat().st_size == 0:
        tmp.unlink(missing_ok=True)
        return None
    return tmp
EOF
```

**4. Templates:**

```bash
cat > ~/surveillance/web/templates/base.html <<'EOF'
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <meta name="referrer" content="no-referrer">
  <link rel="icon" href="data:,">
  <title>{{ title }} · {{ camera_name }}</title>
  <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
  <script src="{{ url_for('static', filename='app.js') }}" defer></script>
  {% block scripts %}{% endblock %}
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

cat > ~/surveillance/web/templates/_timeline.html <<'EOF'
{% macro timeline(tl) %}
<div class="timeline-wrap">
  <svg class="timeline" viewBox="0 0 1440 40" preserveAspectRatio="none" role="img"
       aria-label="Recordings and motion events over the day">
    <rect class="tl-bg" x="0" y="8" width="1440" height="24"></rect>
    {% for s in tl.spans %}<rect class="tl-rec" x="{{ s.x }}" y="8" width="{{ s.w }}" height="24"></rect>{% endfor %}
    {% for e in tl.events %}
      {% if e.url %}<a href="{{ e.url }}">{% endif %}
      <rect class="tl-event" x="{{ e.x }}" y="2" width="{{ e.w }}" height="36"><title>{{ e.label }}</title></rect>
      {% if e.url %}</a>{% endif %}
    {% endfor %}
    {% if tl.now_x is not none %}<rect class="tl-now" x="{{ tl.now_x }}" y="0" width="1.5" height="40"></rect>{% endif %}
  </svg>
  <div class="tl-hours" aria-hidden="true">
    <span>00:00</span><span>06:00</span><span>12:00</span><span>18:00</span><span>24:00</span>
  </div>
  <p class="tl-legend"><span class="key key-rec"></span> recorded <span class="key key-event"></span> motion (click to play)
    <span class="key key-now"></span> now</p>
</div>
{% endmacro %}
EOF

cat > ~/surveillance/web/templates/recordings.html <<'EOF'
{% extends "base.html" %}
{% from "_timeline.html" import timeline %}
{% block content %}
<section class="panel">
  <div class="day-nav">
    <a class="button" href="{{ url_for('rec.recordings', date=prev_day.isoformat()) }}">&larr; {{ prev_day.strftime('%d %b') }}</a>
    <h2 class="day-title">{{ filters.day.strftime('%A %d %B %Y') }}</h2>
    <a class="button" href="{{ url_for('rec.recordings', date=next_day.isoformat()) }}">{{ next_day.strftime('%d %b') }} &rarr;</a>
  </div>
  {{ timeline(day_timeline) }}
</section>

<section class="panel">
  <h2>Search</h2>
  <form class="filters" method="get" action="{{ url_for('rec.recordings') }}">
    <label>Date <input type="date" name="date" value="{{ filters.day.isoformat() }}"></label>
    <label>From <input type="time" name="from" value="{{ filters.time_from }}"></label>
    <label>To <input type="time" name="to" value="{{ filters.time_to }}"></label>
    <label>Segment <input type="text" name="segment" value="{{ filters.segment }}" placeholder="e.g. 06-05-00Z" maxlength="40"></label>
    <label class="check"><input type="checkbox" name="motion" value="1" {% if filters.motion_only %}checked{% endif %}> Motion only</label>
    <button class="button primary" type="submit">Search</button>
    <a class="button" href="{{ url_for('rec.recordings') }}">Reset</a>
  </form>
  {% if error %}<p class="error" role="alert">{{ error }}</p>{% endif %}
  {% if days %}
  <p class="muted">Days with recordings:
    {% for d in days %}<a href="{{ url_for('rec.recordings', date=d) }}">{{ d }}</a>{% if not loop.last %} · {% endif %}{% endfor %}
  </p>
  {% endif %}
</section>

<section class="panel">
  <h2>{{ items|length }} recording{{ '' if items|length == 1 else 's' }}{% if all_days %} (all days){% endif %}</h2>
  {% if items|length >= max_rows %}<p class="muted">Showing the first {{ max_rows }} only; narrow the search.</p>{% endif %}
  {% if items %}
  <div class="table-wrap">
    <table class="list">
      <thead><tr><th>Time</th><th>Length</th><th>Size</th><th>Flags</th><th></th></tr></thead>
      <tbody>
      {% for r in items %}
        <tr>
          <td><a href="{{ url_for('rec.recording', rid=r.id) }}">{% if all_days %}{{ r.start.strftime('%Y-%m-%d ') }}{% endif %}{{ r.start.strftime('%H:%M:%S') }} &ndash; {{ r.end.strftime('%H:%M:%S') if r.end else '?' }}</a></td>
          <td>{{ r.duration }}</td>
          <td>{{ '%.1f'|format(r.size_mb) }} MB</td>
          <td>
            {% if r.has_motion %}<span class="badge motion">motion</span>{% endif %}
            {% if r.recovered %}<span class="badge warn">recovered</span>{% endif %}
            {% if r.clock_unsynced %}<span class="badge warn">clock not synced</span>{% endif %}
          </td>
          <td class="actions">
            <a class="button small primary" href="{{ url_for('rec.recording', rid=r.id) }}">Play</a>
            <a class="button small" href="{{ url_for('rec.download', rid=r.id) }}">Download</a>
          </td>
        </tr>
      {% endfor %}
      </tbody>
    </table>
  </div>
  {% else %}
  <p class="muted">No recordings match.</p>
  {% endif %}
</section>
{% endblock %}
EOF

cat > ~/surveillance/web/templates/events.html <<'EOF'
{% extends "base.html" %}
{% from "_timeline.html" import timeline %}
{% block content %}
<section class="panel">
  <div class="day-nav">
    <a class="button" href="{{ url_for('rec.events', date=prev_day.isoformat()) }}">&larr; {{ prev_day.strftime('%d %b') }}</a>
    <h2 class="day-title">{{ day.strftime('%A %d %B %Y') }}</h2>
    <a class="button" href="{{ url_for('rec.events', date=next_day.isoformat()) }}">{{ next_day.strftime('%d %b') }} &rarr;</a>
  </div>
  {{ timeline(day_timeline) }}
  {% if error %}<p class="error" role="alert">{{ error }}</p>{% endif %}
  {% if days %}
  <p class="muted">Days with motion:
    {% for d in days %}<a href="{{ url_for('rec.events', date=d) }}">{{ d }}</a>{% if not loop.last %} · {% endif %}{% endfor %}
  </p>
  {% endif %}
</section>

<section class="panel">
  <h2>{{ items|length }} motion event{{ '' if items|length == 1 else 's' }} (newest first)</h2>
  {% if items %}
  <ul class="events">
    {% for e in items %}
    <li>
      <span class="event-time">{{ e.start.strftime('%H:%M:%S') }}</span>
      <span class="event-text">Motion detected · {{ e.duration }}{% if e.ongoing %} (ongoing){% endif %} · peak {{ '%.1f'|format(e.peak) }}% of the frame</span>
      {% if e.url %}
        <a class="button small primary" href="{{ e.url }}">Play</a>
      {% else %}
        <span class="muted">{{ 'still recording' if e.ongoing else 'footage deleted' }}</span>
      {% endif %}
    </li>
    {% endfor %}
  </ul>
  {% else %}
  <p class="muted">No motion on this day.</p>
  {% endif %}
</section>
{% endblock %}
EOF

cat > ~/surveillance/web/templates/player.html <<'EOF'
{% extends "base.html" %}
{% block scripts %}<script src="{{ url_for('static', filename='player.js') }}" defer></script>{% endblock %}
{% block content %}
<section class="panel">
  <div class="day-nav">
    {% if prev_url %}<a class="button" href="{{ prev_url }}">&larr; Previous</a>{% else %}<span></span>{% endif %}
    <h2 class="day-title">{{ rec.start.strftime('%a %d %b %Y, %H:%M:%S') }} &ndash; {{ rec.end.strftime('%H:%M:%S') if rec.end else '?' }}</h2>
    {% if next_url %}<a class="button" href="{{ next_url }}">Next &rarr;</a>{% else %}<span></span>{% endif %}
  </div>
  <video id="player" class="player" controls preload="metadata" playsinline
         src="{{ url_for('rec.video', rid=rec.id) }}"
         data-start="{{ '%.0f'|format(start_at) }}" data-next="{{ next_url or '' }}"></video>
  <div class="player-actions">
    <a class="button primary" href="{{ url_for('rec.download', rid=rec.id) }}">Download MP4</a>
    <label class="check"><input type="checkbox" id="autoplay-next"> Play the next segment automatically</label>
    <a class="button" href="{{ url_for('rec.recordings', date=rec.start.date().isoformat()) }}">Back to the day</a>
  </div>
</section>

{% if events %}
<section class="panel">
  <h2>Motion in this recording</h2>
  <ul class="events">
    {% for e in events %}
    <li>
      <span class="event-time">{{ e.start.strftime('%H:%M:%S') }}</span>
      <span class="event-text">Motion · {{ e.duration }}</span>
      <button class="button small primary" type="button" data-seek="{{ e.offset }}">Jump to {{ e.offset // 60 }}:{{ '%02d'|format(e.offset % 60) }}</button>
    </li>
    {% endfor %}
  </ul>
</section>
{% endif %}

<section class="panel">
  <h2>Details</h2>
  <dl class="facts">
    <div><dt>Length</dt><dd>{{ rec.duration }}</dd></div>
    <div><dt>Size</dt><dd>{{ '%.1f'|format(rec.size_mb) }} MB</dd></div>
    <div><dt>Motion</dt><dd>{{ 'yes' if rec.has_motion else 'no' }}</dd></div>
    <div><dt>Status</dt><dd>{{ 'recovered after a crash or power cut' if rec.recovered else 'complete' }}</dd></div>
    <div><dt>Clock</dt><dd>{{ 'NOT synchronised when recorded; times may be wrong' if rec.clock_unsynced else 'OK' }}</dd></div>
    <div><dt>File</dt><dd>{{ rec.rel_path }}</dd></div>
  </dl>
</section>
{% endblock %}
EOF
```

**5. Player script, and styles added to the end of `style.css`:**

```bash
cat > ~/surveillance/web/static/player.js <<'EOF'
"use strict";

// Playback helpers: start position, "jump to motion" buttons and automatic next segment.
(() => {
  const video = document.getElementById("player");
  if (!video) return;

  const start = Number(video.dataset.start || 0);
  if (start > 0) {
    video.addEventListener("loadedmetadata", () => { video.currentTime = start; }, { once: true });
  }

  document.querySelectorAll("[data-seek]").forEach((button) => {
    button.addEventListener("click", () => {
      video.currentTime = Number(button.dataset.seek);
      video.play().catch(() => {});
    });
  });

  const auto = document.getElementById("autoplay-next");
  if (auto) {
    auto.checked = localStorage.getItem("autoplayNext") === "1";
    auto.addEventListener("change", () => localStorage.setItem("autoplayNext", auto.checked ? "1" : "0"));
  }
  video.addEventListener("ended", () => {
    if (video.dataset.next && auto && auto.checked) window.location.href = `${video.dataset.next}?autoplay=1`;
  });
  if (new URLSearchParams(window.location.search).get("autoplay") === "1") video.play().catch(() => {});
})();
EOF

cat >> ~/surveillance/web/static/style.css <<'EOF'

/* ---- Phase 9: recordings, events, player ---- */
.button {
  display: inline-flex;
  align-items: center;
  gap: .35rem;
  padding: .45rem .85rem;
  border: 1px solid var(--border);
  border-radius: 8px;
  background: var(--panel-2);
  color: var(--text);
  font: inherit;
  font-size: .9rem;
  text-decoration: none;
  cursor: pointer;
  white-space: nowrap;
}
.button:hover { border-color: var(--accent); }
.button.primary { background: #1f3b63; border-color: #2d5a94; }
.button.small { padding: .25rem .6rem; font-size: .8rem; }

a { color: var(--accent); }

.day-nav {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: .75rem;
  margin-bottom: .75rem;
}
.day-title { margin: 0; text-align: center; text-transform: none; letter-spacing: 0; font-size: 1rem; color: var(--text); }

.timeline { width: 100%; height: 48px; display: block; border-radius: 6px; }
.tl-bg { fill: #11151a; }
.tl-rec { fill: #24527f; }
.tl-event { fill: var(--warn); opacity: .9; cursor: pointer; }
.tl-event:hover { fill: #ffd27a; }
.tl-now { fill: var(--bad); }
.tl-hours { display: flex; justify-content: space-between; color: var(--muted); font-size: .75rem; margin-top: .25rem; }
.tl-legend { color: var(--muted); font-size: .8rem; margin: .4rem 0 0; display: flex; align-items: center; gap: .4rem; flex-wrap: wrap; }
.key { display: inline-block; width: .9rem; height: .6rem; border-radius: 2px; }
.key-rec { background: #24527f; }
.key-event { background: var(--warn); margin-left: .6rem; }
.key-now { background: var(--bad); width: .2rem; margin-left: .6rem; }

.filters { display: flex; flex-wrap: wrap; gap: .75rem; align-items: flex-end; }
.filters label { display: grid; gap: .25rem; font-size: .8rem; color: var(--muted); }
.filters input[type="date"], .filters input[type="time"], .filters input[type="text"] {
  padding: .4rem .5rem;
  border: 1px solid var(--border);
  border-radius: 8px;
  background: var(--bg);
  color: var(--text);
  font: inherit;
}
label.check { display: inline-flex; align-items: center; gap: .4rem; color: var(--text); font-size: .9rem; }
.error { color: var(--bad); margin: .6rem 0 0; }

.table-wrap { overflow-x: auto; }
table.list { width: 100%; border-collapse: collapse; font-size: .9rem; }
table.list th { text-align: left; color: var(--muted); font-weight: 500; padding: .4rem .5rem; border-bottom: 1px solid var(--border); }
table.list td { padding: .45rem .5rem; border-bottom: 1px solid var(--border); white-space: nowrap; }
table.list td.actions { text-align: right; display: flex; gap: .4rem; justify-content: flex-end; }
table.list tr:hover td { background: #1a1f26; }

.badge { display: inline-block; padding: .05rem .45rem; border-radius: 99px; font-size: .75rem; border: 1px solid var(--border); }
.badge.motion { color: var(--warn); border-color: #6b5320; }
.badge.warn { color: var(--bad); border-color: #6b2b2a; }

ul.events { list-style: none; margin: 0; padding: 0; }
ul.events li { display: flex; align-items: center; gap: .75rem; padding: .5rem 0; border-bottom: 1px solid var(--border); flex-wrap: wrap; }
.event-time { font-variant-numeric: tabular-nums; font-weight: 650; min-width: 5rem; }
.event-text { flex: 1; color: var(--muted); }

.player { width: 100%; max-height: 70vh; background: #000; border-radius: 8px; display: block; }
.player-actions { display: flex; flex-wrap: wrap; gap: .75rem; align-items: center; margin-top: .75rem; }

main > * { min-width: 0; }

@media (max-width: 600px) {
  .day-nav { flex-wrap: wrap; justify-content: center; }
  table.list td { white-space: normal; }
  table.list th:nth-child(3), table.list td:nth-child(3) { display: none; }
  table.list td.actions { flex-direction: column; align-items: stretch; }
}
EOF
```

## Test procedure

**1. Restart only the web interface.** In window 2, press Ctrl+C and run `python3 -m app.web_main` again. The recorder in window 1 can keep running; nothing in it changed. Keep the laptop tunnel open, or reopen it with `ssh -N -L 8080:127.0.0.1:8080 ysak@ysak.local`.

**2. Recordings page** at **http://localhost:8080/recordings**:
- The timeline shows today's blue recording bars and yellow motion markers.
- Tick **Motion only** and click Search; only segments with motion remain.
- Try a time range, e.g. From 14:00 To 14:10.
- Type part of a segment name, e.g. `06-05`, and **clear the date**. It should find matching segments across all days.
- Type `<script>` into Segment and search. You should get an error message, and nothing else should happen.

**3. Play a recording.**
- Drag the seek bar around. It should jump immediately.
- Try **Next →** and **← Previous**.
- Tick **Play the next segment automatically**, seek to near the end, and check the next segment starts on its own.

**4. Motion Events page.**
- Click **Play** on an event. It should start about 5 s before you appear.
- Click a **yellow marker** on the timeline. It should do the same.
- On a recording with motion, use the **Jump to …** button.

**5. Download.** Click **Download** on any recording, then open the file in **VLC**. It should play and seek normally, which is the case that showed a still image back in Phase 3.

**6. Security checks, on the Pi:**

```bash
curl -s -o /dev/null -w "%{http_code}\n" "http://127.0.0.1:8080/recordings/../../../etc/passwd"   # 404
curl -s -o /dev/null -w "%{http_code}\n" -r 0-99 http://127.0.0.1:8080/recordings/1/video          # 206
ls /tmp/surveillance-download-* 2>/dev/null | wc -l                                               # 0
```

The first command may print `400`, since recording `1` may have been deleted by the storage limit. What matters is that it doesn't print `200`. If the second prints `404`, replace `1` with an ID taken from a Play link.

**About your phone:** it can't use your laptop's tunnel. Phone access arrives with Tailscale in Phase 12. Until then you can preview the phone layout in the laptop browser's developer tools, using its responsive or device mode.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `Service Unavailable` on Recordings | The database doesn't exist yet. Start the recorder once. |
| The video shows 0:00 and won't play | Tell me your browser and version. Also check `logs/web.log`. |
| The download is a large file that VLC shows as a still image | The conversion was skipped, because another one was already running or the file is over 300 MB. Try again after a moment. |
| Times look shifted by 8 hours | Check the Pi's time zone with `timedatectl`. The pages use the Pi's local time zone. |

**Please send me:** a screenshot of the Recordings page and of the Motion Events page, whether playback and seeking worked in your browser, and whether the downloaded file plays in VLC.

Phase 10 then brings the **live view**: low-latency video in the browser that only runs while someone is watching.
