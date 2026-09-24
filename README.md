Phase 7 (the SQLite database) is ready for you to run on the Pi. It passed all my tests here with the fake camera but hasn't run on real hardware yet.

## Phase 6 results

- **Emergency storage pause:** confirmed on the Pi. You got `Storage critical … PAUSED`, then `Recording paused`, and no new segments.
- **Motion detection:** works. Events are logged and segments are marked `motion=yes` or `motion=no`.
- **Motion right after start:** in both runs, motion started 2–4 s after launch. If you weren't moving in front of the camera, that was auto-exposure still adjusting. In step 0 it also ran for 80.8 s with a 57.4 % peak, which is more likely you in front of the camera; the settle change won't affect that. The detector now waits **3 s** after start instead of 1 s (included in this update).
- **CPU:** 12.4 % and 24 % of one core in two 5-second samples. That's too short to judge, so step 6 below measures a 60-second average.
- **Memory:** 190 MB, in line with my estimate.

## What Phase 7 adds

**The database index.** `database/surveillance.db` is an index of every recording and motion event. The files on disk remain the source of truth; the database makes browsing, searching and the timeline fast. The web app in Phase 8 will read it.

| Table | Contents |
|---|---|
| `cameras` | Camera ID and name (renaming the camera later keeps all recordings attached to it) |
| `recordings` | File path, start, end, duration, size, frames, `complete` or `recovered`, motion flag, clock-synced flag |
| `motion_events` | Start, end, duration, peak area |
| `motion_event_recordings` | Links between events and recordings. An event that crosses a segment boundary is linked to both files |
| `v_motion_events` | A ready-made view in your original sketch's layout: one row per event with its `recording_file` |

All times are stored as UTC milliseconds. Phase 8 converts them to your local time.

**Safety:**
- **No waiting on the database.** One background writer thread does all database writes, so the camera, encoder and motion threads never wait for it. If a database error happens, it's logged and **recording carries on**.
- **Durable writes.** SQLite's WAL mode with `synchronous=FULL` means a saved change survives a power cut.
- **At every start:**
  - an integrity check runs;
  - a damaged file is moved aside as `surveillance.db.corrupt-<time>` and the newest good backup is restored;
  - motion events left open by a crash are closed;
  - the index is **reconciled against the recordings folder**: files without a row are added, and rows whose file is gone are removed.
- **Daily backup** to `database/backups/`, keeping the last 3.
- **Old events:** events whose footage has been deleted are kept for 90 days (`motion.keep_events_days`).

The "prefer motion footage when deleting" retention mode from the design isn't included yet. It's the only planned storage feature still missing, and I'll add it with the Storage settings page in Phase 8. For now, `oldest_first` plus the optional `max_age_days` is the retention behaviour.

## What I tested here

**Database on its own:**
- An event that spanned 16:00 and 16:05 was linked to both segments, including the one that finished while the event was still ongoing.
- An event left open at a simulated crash was closed on restart.
- Reconcile added a `.recovered.mp4` (with its real 12.0 s length from `ffprobe`) and removed the row for a deleted file.
- **Deliberate corruption:** I overwrote 4 KB of the database file. It was moved aside, the backup restored, the integrity check came back `ok`, and the index was rebuilt from the files.
- Renaming the camera kept all recordings on camera ID 1.

**End to end with the fake camera:**
- The 11.8 s event was linked to the right segment; the next segment had `motion=no`.
- The report tool read the database safely both while the recorder was writing and after it stopped.
- After a `kill -9` and restart, the index still matched the folder.

**Update script:** it turns your 0.6.0 files into files identical to the tested ones, and aborts safely if run twice.

## Install (optional)

This gives you the `sqlite3` command for looking inside the database yourself:

```bash
sudo apt install -y sqlite3
```

## Update the files

**1. Apply the update.** It changes `__init__.py`, `config.py`, `storage_manager.py`, `motion_detector.py` and `main.py`, checks every piece of code it replaces, and makes `.bak` backups.

```bash
cd ~/surveillance
cat > update_to_0_7_0.py <<'EOF'
#!/usr/bin/env python3
"""Update the surveillance project from 0.6.0 to 0.7.0 (run from ~/surveillance)."""
import shutil, sys
from pathlib import Path

EDITS = [
    ('app/__init__.py',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.6.0"\n',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.7.0"\n'),
    ('app/config.py',
     '    trigger_frames: int = 3\n    analysis_fps: float = 5.0\n\n    def validate(self) -> None:\n        _check_range("motion.sensitivity", self.sensitivity, 1, 100)\n        _check_range("motion.min_area_percent", self.min_area_percent, 0.05, 50.0)\n',
     '    trigger_frames: int = 3\n    analysis_fps: float = 5.0\n    keep_events_days: int = 90\n\n    def validate(self) -> None:\n        _check_range("motion.keep_events_days", self.keep_events_days, 0, 3650)\n        _check_range("motion.sensitivity", self.sensitivity, 1, 100)\n        _check_range("motion.min_area_percent", self.min_area_percent, 0.05, 50.0)\n'),
    ('app/config.py',
     'class PathSettings:\n    recordings_dir: str = "recordings"\n    log_dir: str = "logs"\n\n    def validate(self) -> None:\n        for name in ("recordings_dir", "log_dir"):\n            value = getattr(self, name)\n            if not value.strip() or "\\x00" in value:\n',
     'class PathSettings:\n    recordings_dir: str = "recordings"\n    database_dir: str = "database"\n    log_dir: str = "logs"\n\n    def validate(self) -> None:\n        for name in ("recordings_dir", "database_dir", "log_dir"):\n            value = getattr(self, name)\n            if not value.strip() or "\\x00" in value:\n'),
    ('app/config.py',
     '    def recordings_dir(self) -> Path:\n        return resolve_path(self.paths.recordings_dir)\n\n    @property\n',
     '    def recordings_dir(self) -> Path:\n        return resolve_path(self.paths.recordings_dir)\n\n    @property\n    def database_dir(self) -> Path:\n        return resolve_path(self.paths.database_dir)\n\n    @property\n'),
    ('app/storage_manager.py',
     '\nclass StorageManager:\n    def __init__(self, settings: Settings, current_segment: Callable[[], Path | None]) -> None:\n        s = settings.storage\n        self.root = settings.recordings_dir\n        self._required_mount = Path(s.required_mount) if s.required_mount else None\n',
     '\nclass StorageManager:\n    def __init__(self, settings: Settings, current_segment: Callable[[], Path | None],\n                 on_deleted: Callable[[list[str]], None] | None = None,\n                 on_recovered: Callable[[str, float, int, float], None] | None = None) -> None:\n        s = settings.storage\n        self._on_deleted = on_deleted\n        self._on_recovered = on_recovered\n        self.root = settings.recordings_dir\n        self._required_mount = Path(s.required_mount) if s.required_mount else None\n'),
    ('app/storage_manager.py',
     '        freed = 0\n        deleted = 0\n        touched_days: set[Path] = set()\n        for f, reason in plan:\n',
     '        freed = 0\n        deleted = 0\n        deleted_paths: list[str] = []\n        touched_days: set[Path] = set()\n        for f, reason in plan:\n'),
    ('app/storage_manager.py',
     '                f.path.unlink()\n            except FileNotFoundError:\n                continue\n            except OSError as exc:\n                log.error("Cannot delete %s: %s", f.rel_path, exc.strerror)\n                continue\n            deleted += 1\n            freed += f.size\n',
     '                f.path.unlink()\n            except FileNotFoundError:\n                deleted_paths.append(f.rel_path)\n                continue\n            except OSError as exc:\n                log.error("Cannot delete %s: %s", f.rel_path, exc.strerror)\n                continue\n            deleted_paths.append(f.rel_path)\n            deleted += 1\n            freed += f.size\n'),
    ('app/storage_manager.py',
     '        if deleted > MAX_INDIVIDUAL_DELETE_LOGS:\n            log.info("Deleted %d segments in total, freeing %.1f GB", deleted, freed / GB)\n        today = datetime.now(timezone.utc).strftime("%Y-%m-%d")\n        for day in touched_days:\n',
     '        if deleted > MAX_INDIVIDUAL_DELETE_LOGS:\n            log.info("Deleted %d segments in total, freeing %.1f GB", deleted, freed / GB)\n        if deleted_paths and self._on_deleted:\n            self._on_deleted(deleted_paths)\n        today = datetime.now(timezone.utc).strftime("%Y-%m-%d")\n        for day in touched_days:\n'),
    ('app/storage_manager.py',
     '        log.warning("Recovered unfinished segment %s -> %s: %.1f s playable%s", f.rel_path, recovered.name,\n                    seconds, f", removed {trimmed / 1024:.0f} KiB of incomplete data" if trimmed else "")\n        return True\n',
     '        log.warning("Recovered unfinished segment %s -> %s: %.1f s playable%s", f.rel_path, recovered.name,\n                    seconds, f", removed {trimmed / 1024:.0f} KiB of incomplete data" if trimmed else "")\n        if self._on_recovered:\n            self._on_recovered(str(recovered.relative_to(self.root)), seconds, new_size, f.start)\n        return True\n'),
    ('app/motion_detector.py',
     'LIGHTING_CHANGE_PERCENT = 60.0\nRELEARN_SECONDS = 1.0\nMAX_EVENT_SECONDS = 3600.0\nCAPTURE_TIMEOUT_S = 2.0\n',
     'LIGHTING_CHANGE_PERCENT = 60.0\nRELEARN_SECONDS = 1.0\nSTARTUP_SETTLE_SECONDS = 3.0   # auto-exposure is still adjusting right after the camera starts\nMAX_EVENT_SECONDS = 3600.0\nCAPTURE_TIMEOUT_S = 2.0\n'),
    ('app/motion_detector.py',
     '    end: float | None = None\n    peak_percent: float = 0.0\n\n    @property\n',
     '    end: float | None = None\n    peak_percent: float = 0.0\n    db_id: int | None = None         # set by the database writer\n\n    @property\n'),
    ('app/motion_detector.py',
     '        if self._background is None:\n            self._background = blurred.astype(np.float32)\n            self._relearn_until = now + RELEARN_SECONDS\n            return FrameScore(0.0, 0.0, False, False)\n        if now < self._relearn_until:\n',
     '        if self._background is None:\n            self._background = blurred.astype(np.float32)\n            self._relearn_until = now + STARTUP_SETTLE_SECONDS\n            return FrameScore(0.0, 0.0, False, False)\n        if now < self._relearn_until:\n'),
    ('app/main.py',
     'from app.clock import ClockMonitor  # noqa: E402\nfrom app.config import DEFAULT_CONFIG_PATH, ConfigError, Settings, load_settings  # noqa: E402\nfrom app.logging_setup import setup_logging  # noqa: E402\nfrom app.recorder import Recorder  # noqa: E402\n',
     'from app.clock import ClockMonitor  # noqa: E402\nfrom app.config import DEFAULT_CONFIG_PATH, ConfigError, Settings, load_settings  # noqa: E402\nfrom app.database import Database  # noqa: E402\nfrom app.logging_setup import setup_logging  # noqa: E402\nfrom app.recorder import Recorder  # noqa: E402\n'),
    ('app/main.py',
     '\n\ndef run(settings: Settings, stop: threading.Event) -> None:\n    clock = ClockMonitor()\n    clock.start()\n    active: list[Recorder] = []\n    storage = StorageManager(settings, lambda: active[0].current_partial_path if active else None)\n    # Recover unfinished segments and free space before the camera starts writing.\n    try:\n',
     '\n\ndef open_database(settings: Settings) -> Database | None:\n    db = Database(settings)\n    try:\n        db.open()\n    except Exception:  # noqa: BLE001 - recording must work even without the index\n        log.exception("Cannot open the database; recording continues without an index")\n        return None\n    db.start()\n    return db\n\n\ndef run(settings: Settings, stop: threading.Event) -> None:\n    clock = ClockMonitor()\n    clock.start()\n    db = open_database(settings)\n    active: list[Recorder] = []\n    storage = StorageManager(settings, lambda: active[0].current_partial_path if active else None,\n                             on_deleted=db.recordings_deleted if db else None,\n                             on_recovered=db.recording_recovered if db else None)\n    # Recover unfinished segments and free space before the camera starts writing.\n    try:\n'),
    ('app/main.py',
     '        log.exception("Initial storage check failed")\n    storage.start()\n    backoff = BACKOFF_INITIAL_S\n    try:\n',
     '        log.exception("Initial storage check failed")\n    storage.start()\n    if db:\n        db.reconcile()\n    backoff = BACKOFF_INITIAL_S\n    try:\n'),
    ('app/main.py',
     '            recorder = Recorder(settings, clock, storage.can_write, storage.location_error)\n            recorder.add_listener(lambda _segment: storage.trigger())\n            active[:] = [recorder]\n            try:\n',
     '            recorder = Recorder(settings, clock, storage.can_write, storage.location_error)\n            recorder.add_listener(lambda _segment: storage.trigger())\n            if db:\n                recorder.add_listener(db.add_segment)\n                recorder.add_motion_listener(db.motion_event)\n            active[:] = [recorder]\n            try:\n'),
    ('app/main.py',
     '        active.clear()\n        storage.stop()\n        clock.stop()\n\n',
     '        active.clear()\n        storage.stop()\n        if db:\n            db.stop()\n        clock.stop()\n\n'),
]

texts = {}
for rel, old, new in EDITS:
    text = texts.setdefault(rel, Path(rel).read_text())
    if text.count(old) != 1:
        sys.exit(f"ABORTED, nothing changed: {rel} does not match the expected 0.6.0 code "
                 f"(found {text.count(old)} matches for:\n{old})")
    texts[rel] = text.replace(old, new)
for rel, text in texts.items():
    shutil.copy2(rel, rel + ".bak")
    Path(rel).write_text(text)
    print(f"updated {rel}  (backup: {rel}.bak)")
print("Update to 0.7.0 complete.")
EOF
python3 update_to_0_7_0.py
```

It should print five `updated …` lines and then `Update to 0.7.0 complete.`

**2. The new database module:**

```bash
cat > ~/surveillance/app/database.py <<'EOF'
"""SQLite index of recordings and motion events.

The recording files on disk are the source of truth; this database is an index that makes
browsing, searching and the event timeline fast. It is written by one background thread,
so the camera, encoder and motion threads never wait for the disk.

Safety:
  * WAL journal + synchronous=FULL: a committed change survives a power cut.
  * At startup: integrity check; a damaged file is moved aside and the newest good backup
    is restored (or a fresh database is created), then the index is rebuilt from the files.
  * A backup copy is made once a day (VACUUM INTO), keeping the last few.
  * Any database error is logged and recording carries on.

All times are stored as UTC milliseconds since 1970 (INTEGER).
"""
from __future__ import annotations

import logging
import queue
import shutil
import sqlite3
import subprocess
import threading
import time
from datetime import datetime, timezone
from pathlib import Path
from typing import Callable

from app.config import Settings
from app.storage_manager import RECOVERED_SUFFIX, scan_segments

log = logging.getLogger("Database")

SCHEMA_VERSION = 1
DB_FILE_NAME = "surveillance.db"
BACKUP_DIR_NAME = "backups"
BACKUPS_KEPT = 3
BACKUP_INTERVAL_S = 24 * 3600
PRUNE_INTERVAL_S = 6 * 3600
IDLE_WAKE_S = 60.0

SCHEMA = """
CREATE TABLE IF NOT EXISTS schema_version (version INTEGER NOT NULL);

CREATE TABLE IF NOT EXISTS cameras (
    id          INTEGER PRIMARY KEY,
    name        TEXT NOT NULL UNIQUE,
    created_at  INTEGER NOT NULL
);

CREATE TABLE IF NOT EXISTS recordings (
    id           INTEGER PRIMARY KEY,
    camera_id    INTEGER NOT NULL REFERENCES cameras(id),
    rel_path     TEXT NOT NULL UNIQUE,          -- relative to recordings_dir
    start_utc    INTEGER NOT NULL,
    end_utc      INTEGER,
    duration_ms  INTEGER,
    size_bytes   INTEGER NOT NULL,
    frames       INTEGER,
    status       TEXT NOT NULL CHECK (status IN ('complete', 'recovered')),
    has_motion   INTEGER NOT NULL DEFAULT 0,
    clock_synced INTEGER,                        -- 1, 0, or NULL if unknown
    created_at   INTEGER NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_recordings_camera_start ON recordings(camera_id, start_utc);

CREATE TABLE IF NOT EXISTS motion_events (
    id             INTEGER PRIMARY KEY,
    camera_id      INTEGER NOT NULL REFERENCES cameras(id),
    start_utc      INTEGER NOT NULL,
    end_utc        INTEGER,                      -- NULL while the event is ongoing
    duration_ms    INTEGER,
    peak_area_pct  REAL,
    created_at     INTEGER NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_motion_camera_start ON motion_events(camera_id, start_utc);

-- An event can span two segments, so events and recordings are linked many-to-many.
CREATE TABLE IF NOT EXISTS motion_event_recordings (
    event_id      INTEGER NOT NULL REFERENCES motion_events(id) ON DELETE CASCADE,
    recording_id  INTEGER NOT NULL REFERENCES recordings(id) ON DELETE CASCADE,
    PRIMARY KEY (event_id, recording_id)
);
CREATE INDEX IF NOT EXISTS idx_links_recording ON motion_event_recordings(recording_id);

-- The event list as originally sketched: one row per event with its first recording file.
CREATE VIEW IF NOT EXISTS v_motion_events AS
SELECT e.id, e.camera_id, e.start_utc, e.end_utc, e.duration_ms, e.peak_area_pct,
       (SELECT r.rel_path FROM motion_event_recordings l JOIN recordings r ON r.id = l.recording_id
         WHERE l.event_id = e.id ORDER BY r.start_utc LIMIT 1) AS recording_file,
       e.created_at
FROM motion_events e;
"""


def now_ms() -> int:
    return int(time.time() * 1000)


def to_ms(epoch_seconds: float) -> int:
    return int(round(epoch_seconds * 1000))


def connect(path: Path, read_only: bool = False) -> sqlite3.Connection:
    if read_only:
        conn = sqlite3.connect(f"file:{path}?mode=ro", uri=True, timeout=5)
    else:
        # Opened by open() in the main thread, then used only by the writer thread.
        conn = sqlite3.connect(path, timeout=5, isolation_level=None, check_same_thread=False)
        conn.execute("PRAGMA journal_mode=WAL")
        conn.execute("PRAGMA synchronous=FULL")
    conn.execute("PRAGMA foreign_keys=ON")
    conn.execute("PRAGMA busy_timeout=5000")
    conn.row_factory = sqlite3.Row
    return conn


def probe_duration_ms(path: Path) -> int | None:
    try:
        result = subprocess.run(["ffprobe", "-v", "error", "-show_entries", "format=duration",
                                 "-of", "csv=p=0", str(path)],
                                capture_output=True, text=True, timeout=30, check=False)
        return int(float(result.stdout.strip()) * 1000)
    except (OSError, subprocess.TimeoutExpired, ValueError):
        return None


class Database:
    def __init__(self, settings: Settings) -> None:
        self.path = settings.database_dir / DB_FILE_NAME
        self.backup_dir = settings.database_dir / BACKUP_DIR_NAME
        self.recordings_dir = settings.recordings_dir
        self._camera_name = settings.camera.name
        self._keep_events_days = settings.motion.keep_events_days
        self._segment_ms = settings.recording.segment_seconds * 1000
        self._conn: sqlite3.Connection | None = None
        self.camera_id = settings.camera.camera_num + 1
        self._queue: queue.Queue[tuple[Callable, tuple] | None] = queue.Queue()
        self._thread = threading.Thread(target=self._run, name="database-writer", daemon=True)
        self._last_backup = 0.0
        self._last_prune = 0.0
        self.healthy = True

    # ---------------------------------------------------------------- lifecycle
    def open(self) -> None:
        """Open (checking, restoring or creating) the database. Call before start()."""
        self.path.parent.mkdir(parents=True, exist_ok=True)
        if self.path.exists() and not self._integrity_ok(self.path):
            self._replace_damaged_database()
        self._conn = connect(self.path)
        self._conn.executescript(SCHEMA)  # idempotent; executescript commits on its own
        with self._transaction():
            row = self._conn.execute("SELECT version FROM schema_version").fetchone()
            if row is None:
                self._conn.execute("INSERT INTO schema_version (version) VALUES (?)", (SCHEMA_VERSION,))
            elif row["version"] > SCHEMA_VERSION:
                raise sqlite3.DatabaseError(f"database schema {row['version']} is newer than this program")
            # The camera number is the stable identity; the name is only a label that may change.
            self._conn.execute("INSERT OR IGNORE INTO cameras (id, name, created_at) VALUES (?, ?, ?)",
                               (self.camera_id, self._camera_name, now_ms()))
            self._conn.execute("UPDATE cameras SET name = ? WHERE id = ?", (self._camera_name, self.camera_id))
            closed = self._conn.execute(
                "UPDATE motion_events SET end_utc = start_utc, duration_ms = 0 WHERE end_utc IS NULL").rowcount
        if closed:
            log.warning("Closed %d motion event(s) left open by a crash or power cut", closed)
        self._last_backup = self._newest_backup_time()
        log.info("Database ready: %s", self.path)

    def start(self) -> None:
        self._thread.start()

    def stop(self) -> None:
        if self._thread.is_alive():
            self._queue.put(None)
            self._thread.join(timeout=60)
        if self._conn is not None:
            self._conn.close()
            self._conn = None

    # ------------------------------------------------------- queued write API
    def reconcile(self) -> None:
        self._submit(self._reconcile)

    def add_segment(self, segment) -> None:
        """Recorder listener: a segment has been completed and renamed to .mp4."""
        self._submit(self._insert_recording, segment.rel_path, to_ms(segment.start_utc.timestamp()),
                     to_ms(segment.end_utc.timestamp()), int(segment.duration_s * 1000),
                     segment.size_bytes, segment.frames, "complete", segment.clock_synced)

    def recording_recovered(self, rel_path: str, seconds: float, size_bytes: int, start: float) -> None:
        """Storage-manager listener: an unfinished segment was repaired."""
        self._submit(self._insert_recording, rel_path, to_ms(start), to_ms(start + seconds),
                     int(seconds * 1000), size_bytes, None, "recovered", None)

    def recordings_deleted(self, rel_paths: list[str]) -> None:
        """Storage-manager listener: these files were deleted."""
        self._submit(self._delete_recordings, list(rel_paths))

    def motion_event(self, kind: str, event) -> None:
        """Motion-detector listener: kind is 'start' or 'end'."""
        if kind == "start":
            self._submit(self._event_started, event)
        else:
            self._submit(self._event_ended, event)

    # ------------------------------------------------------------ internals
    def _submit(self, fn: Callable, *args) -> None:
        self._queue.put((fn, args))

    def _run(self) -> None:
        while True:
            try:
                item = self._queue.get(timeout=IDLE_WAKE_S)
            except queue.Empty:
                item = ()
            if item is None:
                return
            try:
                if item:
                    fn, args = item
                    fn(*args)
                self._maintenance()
                if not self.healthy:
                    log.info("Database writes are working again")
                    self.healthy = True
            except sqlite3.Error as exc:
                if self.healthy:
                    log.error("Database error (recording continues; the index is rebuilt at the next "
                              "start): %s", exc)
                self.healthy = False
            except Exception:  # noqa: BLE001 - the writer thread must survive
                log.exception("Unexpected error in the database writer")

    def _transaction(self):
        conn = self._conn

        class _Tx:
            def __enter__(self_inner):
                conn.execute("BEGIN IMMEDIATE")
                return conn

            def __exit__(self_inner, exc_type, exc, tb):
                conn.execute("ROLLBACK" if exc_type else "COMMIT")
                return False
        return _Tx()

    def _insert_recording(self, rel_path: str, start_ms: int, end_ms: int | None, duration_ms: int | None,
                          size_bytes: int, frames: int | None, status: str, clock_synced: bool | None) -> None:
        with self._transaction() as conn:
            conn.execute(
                "INSERT INTO recordings (camera_id, rel_path, start_utc, end_utc, duration_ms, size_bytes, frames,"
                " status, clock_synced, created_at) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)"
                " ON CONFLICT(rel_path) DO UPDATE SET start_utc = excluded.start_utc, end_utc = excluded.end_utc,"
                " duration_ms = excluded.duration_ms, size_bytes = excluded.size_bytes, frames = excluded.frames,"
                " status = excluded.status, clock_synced = excluded.clock_synced",
                (self.camera_id, rel_path, start_ms, end_ms, duration_ms, size_bytes, frames, status,
                 None if clock_synced is None else int(clock_synced), now_ms()))
            recording_id = conn.execute("SELECT id FROM recordings WHERE rel_path = ?", (rel_path,)).fetchone()["id"]
            self._link_recording(conn, recording_id, start_ms, end_ms if end_ms is not None else start_ms)

    def _link_recording(self, conn: sqlite3.Connection, recording_id: int, start_ms: int, end_ms: int) -> None:
        conn.execute(
            "INSERT OR IGNORE INTO motion_event_recordings (event_id, recording_id)"
            " SELECT id, ? FROM motion_events WHERE camera_id = ? AND start_utc <= ?"
            " AND (end_utc IS NULL OR end_utc >= ?)", (recording_id, self.camera_id, end_ms, start_ms))
        conn.execute("UPDATE recordings SET has_motion = EXISTS (SELECT 1 FROM motion_event_recordings"
                     " WHERE recording_id = ?) WHERE id = ?", (recording_id, recording_id))

    def _delete_recordings(self, rel_paths: list[str]) -> None:
        with self._transaction() as conn:
            conn.executemany("DELETE FROM recordings WHERE rel_path = ?", [(p,) for p in rel_paths])

    def _event_started(self, event) -> None:
        with self._transaction() as conn:
            cursor = conn.execute(
                "INSERT INTO motion_events (camera_id, start_utc, peak_area_pct, created_at) VALUES (?, ?, ?, ?)",
                (self.camera_id, to_ms(event.start), round(event.peak_percent, 2), now_ms()))
            event.db_id = cursor.lastrowid

    def _event_ended(self, event) -> None:
        if getattr(event, "db_id", None) is None:
            return
        start_ms, end_ms = to_ms(event.start), to_ms(event.end)
        with self._transaction() as conn:
            conn.execute("UPDATE motion_events SET end_utc = ?, duration_ms = ?, peak_area_pct = ? WHERE id = ?",
                         (end_ms, end_ms - start_ms, round(event.peak_percent, 2), event.db_id))
            rows = conn.execute("SELECT id FROM recordings WHERE camera_id = ? AND start_utc <= ?"
                                " AND end_utc >= ?", (self.camera_id, end_ms, start_ms)).fetchall()
            for row in rows:
                conn.execute("INSERT OR IGNORE INTO motion_event_recordings (event_id, recording_id)"
                             " VALUES (?, ?)", (event.db_id, row["id"]))
                conn.execute("UPDATE recordings SET has_motion = 1 WHERE id = ?", (row["id"],))

    def _reconcile(self) -> None:
        """Make the index match the files on disk (files are the source of truth)."""
        on_disk = {f.rel_path: f for f in scan_segments(self.recordings_dir) if not f.partial}
        in_db = {row["rel_path"] for row in self._conn.execute(
            "SELECT rel_path FROM recordings WHERE camera_id = ?", (self.camera_id,))}
        stale = sorted(in_db - set(on_disk))
        if stale:
            self._delete_recordings(stale)
        missing = sorted(set(on_disk) - in_db)
        for rel_path in missing:
            f = on_disk[rel_path]
            duration_ms = probe_duration_ms(f.path) or self._segment_ms
            start_ms = to_ms(f.start)
            status = "recovered" if rel_path.endswith(RECOVERED_SUFFIX) else "complete"
            self._insert_recording(rel_path, start_ms, start_ms + duration_ms, duration_ms,
                                   f.path.stat().st_size, None, status, None)
        if stale or missing:
            log.info("Index reconciled with the recordings folder: %d added, %d removed", len(missing), len(stale))
        else:
            log.info("Index matches the recordings folder (%d recordings)", len(on_disk))

    def _maintenance(self) -> None:
        now = time.time()
        if now - self._last_prune >= PRUNE_INTERVAL_S:
            self._last_prune = now
            if self._keep_events_days:
                cutoff = now_ms() - self._keep_events_days * 86_400_000
                with self._transaction() as conn:
                    pruned = conn.execute(
                        "DELETE FROM motion_events WHERE camera_id = ? AND start_utc < ? AND id NOT IN"
                        " (SELECT event_id FROM motion_event_recordings)", (self.camera_id, cutoff)).rowcount
                if pruned:
                    log.info("Removed %d motion event(s) older than %d days", pruned, self._keep_events_days)
        if now - self._last_backup >= BACKUP_INTERVAL_S:
            self._last_backup = now
            self._backup()

    def _backup(self) -> None:
        self.backup_dir.mkdir(parents=True, exist_ok=True)
        stamp = datetime.now(timezone.utc).strftime("%Y%m%dT%H%M%SZ")
        target = self.backup_dir / f"surveillance-{stamp}.db"
        tmp = target.with_name(target.name + ".tmp")
        tmp.unlink(missing_ok=True)
        self._conn.execute("VACUUM INTO ?", (str(tmp),))
        tmp.replace(target)
        for old in sorted(self.backup_dir.glob("surveillance-*.db"))[:-BACKUPS_KEPT]:
            old.unlink(missing_ok=True)
        log.info("Database backup written: %s", target.name)

    def _newest_backup_time(self) -> float:
        backups = sorted(self.backup_dir.glob("surveillance-*.db"))
        return backups[-1].stat().st_mtime if backups else 0.0

    @staticmethod
    def _integrity_ok(path: Path) -> bool:
        try:
            conn = sqlite3.connect(path, timeout=5)
            try:
                return conn.execute("PRAGMA quick_check").fetchone()[0] == "ok"
            finally:
                conn.close()
        except sqlite3.Error:
            return False

    def _replace_damaged_database(self) -> None:
        stamp = datetime.now(timezone.utc).strftime("%Y%m%dT%H%M%SZ")
        for suffix in ("", "-wal", "-shm"):
            part = Path(str(self.path) + suffix)
            if part.exists():
                part.rename(Path(f"{self.path}.corrupt-{stamp}{suffix}"))
        log.error("Database failed its integrity check; moved aside as %s.corrupt-%s", self.path.name, stamp)
        for backup in sorted(self.backup_dir.glob("surveillance-*.db"), reverse=True):
            if self._integrity_ok(backup):
                shutil.copy2(backup, self.path)
                log.warning("Restored the database from backup %s; the index will be rebuilt from the files",
                            backup.name)
                return
        log.warning("No usable backup; starting a new database and rebuilding it from the files")
EOF
```

**3. The report tool.** It's read-only, so it's safe to run while the recorder is running.

```bash
cat > ~/surveillance/tools/phase7_db_report.py <<'EOF'
#!/usr/bin/env python3
"""Phase 7: read-only view of the recordings/motion database (safe while the recorder runs).

    python3 ~/surveillance/tools/phase7_db_report.py [--date YYYY-MM-DD] [--motion-only]

Shows an integrity check, totals, and for one day (your local time zone) a timeline,
the recordings and the motion events with the recording files they are linked to.
"""
from __future__ import annotations

import argparse
import sqlite3
import sys
from datetime import date, datetime, time as dtime, timedelta
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent.parent))

from app.config import DEFAULT_CONFIG_PATH, ConfigError, load_settings  # noqa: E402
from app.database import DB_FILE_NAME, connect  # noqa: E402

TIMELINE_SLOTS = 48  # 30 minutes each


def local(ms: int | None) -> datetime | None:
    return datetime.fromtimestamp(ms / 1000).astimezone() if ms is not None else None


def fmt_duration(ms: int | None) -> str:
    if ms is None:
        return "?"
    seconds = ms / 1000
    return f"{seconds / 60:.1f} min" if seconds >= 90 else f"{seconds:.1f} s"


def main() -> int:
    parser = argparse.ArgumentParser(description="Phase 7: database report (read-only)")
    parser.add_argument("--config", type=Path, default=DEFAULT_CONFIG_PATH)
    parser.add_argument("--date", help="local date YYYY-MM-DD (default: today)")
    parser.add_argument("--motion-only", action="store_true", help="only list recordings with motion")
    args = parser.parse_args()
    try:
        settings, _ = load_settings(args.config, create_if_missing=False)
    except ConfigError as exc:
        print(f"Configuration error: {exc}", file=sys.stderr)
        return 2
    path = settings.database_dir / DB_FILE_NAME
    if not path.exists():
        print(f"No database yet at {path}. Start the recorder once first.")
        return 1

    conn = connect(path, read_only=True)
    check = conn.execute("PRAGMA quick_check").fetchone()[0]
    totals = conn.execute(
        "SELECT COUNT(*) AS n, COALESCE(SUM(size_bytes), 0) AS bytes, COALESCE(SUM(duration_ms), 0) AS ms,"
        " SUM(has_motion) AS motion, SUM(status = 'recovered') AS recovered,"
        " MIN(start_utc) AS first, MAX(start_utc) AS last FROM recordings").fetchone()
    events_total = conn.execute("SELECT COUNT(*) FROM motion_events").fetchone()[0]
    print(f"Database        : {path}  (integrity: {check})")
    print(f"Recordings      : {totals['n']} ({totals['motion'] or 0} with motion, "
          f"{totals['recovered'] or 0} recovered), {totals['bytes'] / 1e9:.2f} GB, "
          f"{totals['ms'] / 3_600_000:.1f} h")
    if totals["first"] is not None:
        print(f"Covering        : {local(totals['first']):%Y-%m-%d %H:%M} to {local(totals['last']):%Y-%m-%d %H:%M} "
              f"(local time)")
    print(f"Motion events   : {events_total}")

    day = date.fromisoformat(args.date) if args.date else datetime.now().astimezone().date()
    day_start = datetime.combine(day, dtime.min).astimezone()
    start_ms = int(day_start.timestamp() * 1000)
    end_ms = int((day_start + timedelta(days=1)).timestamp() * 1000)

    recordings = conn.execute(
        "SELECT * FROM recordings WHERE start_utc < ? AND COALESCE(end_utc, start_utc) >= ? ORDER BY start_utc",
        (end_ms, start_ms)).fetchall()
    events = conn.execute(
        "SELECT * FROM motion_events WHERE start_utc >= ? AND start_utc < ? ORDER BY start_utc",
        (start_ms, end_ms)).fetchall()

    slot_ms = (end_ms - start_ms) // TIMELINE_SLOTS
    line = ["·"] * TIMELINE_SLOTS
    for r in recordings:
        first = max(0, (r["start_utc"] - start_ms) // slot_ms)
        last = min(TIMELINE_SLOTS - 1, ((r["end_utc"] or r["start_utc"]) - start_ms) // slot_ms)
        for i in range(first, last + 1):
            if line[i] == "·":
                line[i] = "─"
    for e in events:
        line[min(TIMELINE_SLOTS - 1, (e["start_utc"] - start_ms) // slot_ms)] = "█"

    print(f"\n{day:%d %B %Y} (local time)   ─ recording   █ motion   · nothing recorded")
    print(f"00:00 {''.join(line)} 24:00")

    print(f"\nMotion events ({len(events)}):")
    for e in events:
        files = [row["rel_path"] for row in conn.execute(
            "SELECT r.rel_path FROM motion_event_recordings l JOIN recordings r ON r.id = l.recording_id"
            " WHERE l.event_id = ? ORDER BY r.start_utc", (e["id"],))]
        ongoing = " (ongoing)" if e["end_utc"] is None else ""
        print(f"  {local(e['start_utc']):%H:%M:%S}  Motion detected  {fmt_duration(e['duration_ms'])}{ongoing}, "
              f"peak {e['peak_area_pct'] or 0:.1f}%  ->  {', '.join(files) or 'footage deleted / not yet indexed'}")

    shown = [r for r in recordings if r["has_motion"] or not args.motion_only]
    print(f"\nRecordings ({len(shown)}{' with motion' if args.motion_only else ''}):")
    for r in shown:
        flags = ("  motion" if r["has_motion"] else "") + ("  recovered" if r["status"] == "recovered" else "") \
            + ("  clock-not-synced" if r["clock_synced"] == 0 else "")
        end = local(r["end_utc"])
        print(f"  {local(r['start_utc']):%H:%M:%S} - {end:%H:%M:%S}  {fmt_duration(r['duration_ms']):>8}  "
              f"{r['size_bytes'] / 1e6:6.1f} MB  {r['rel_path']}{flags}" if end else f"  {r['rel_path']}")
    return 0 if check == "ok" else 1


if __name__ == "__main__":
    sys.exit(main())
EOF
```

## Test procedure

Run everything as `ysak` from `~/surveillance`. You'll need two SSH windows for steps 2 and 4.

**1. First start.** Use 60-second segments for testing:

```bash
python3 tools/set_setting.py recording.segment_seconds 60
python3 -m app.main
```

The first lines should include `Database ready`, `Index reconciled … N added` (your existing test recordings get indexed) and `Database backup written`.

**2. Record for about 5 minutes and walk past twice.** Try to have one walk-through **span a minute boundary**, for example from :50 to :10 on the clock. While it runs, look at the report from the second window:

```bash
python3 tools/phase7_db_report.py
```

Then stop the recorder with Ctrl+C and run the report again.

**3. Optional: look inside with SQL.**

```bash
sqlite3 -readonly database/surveillance.db \
  "SELECT id, datetime(start_utc/1000,'unixepoch','localtime'), duration_ms/1000.0, recording_file FROM v_motion_events;"
```

**4. Crash during motion.** Start the recorder, walk in front of the camera, and **while you're still moving** kill it from the second window:

```bash
pkill -9 -f "^python3 -m app.main"
```

Start it again. Expect `Closed 1 motion event(s) left open by a crash or power cut`, then within about 60–90 s a `Recovered unfinished segment …` line. Stop it, and the report should show that segment marked `recovered`.

**5. Database corruption.** Stop the recorder, deliberately damage the database, and start again:

```bash
dd if=/dev/urandom of=database/surveillance.db bs=1 seek=100 count=4000 conv=notrunc
python3 -m app.main
```

Expect `Database failed its integrity check; moved aside …`, then `Restored the database from backup …`, then `Index reconciled …`. Stop it and run the report: it should say `integrity: ok` and list the recordings again. Motion events recorded after the backup are lost, which is expected; the recordings themselves are never lost.

**6. CPU over 60 seconds.** With the recorder running, run this in the second window and wait 60 s. The second line is the 60-second average:

```bash
top -b -n 2 -d 60 -p "$(pgrep -f '^python3 -m app.main')" | grep python3
```

**7. Go back to 5-minute segments:**

```bash
python3 tools/set_setting.py recording.segment_seconds 300
```

## Expected output

The report should look roughly like this:

```text
Database        : /home/ysak/surveillance/database/surveillance.db  (integrity: ok)
Recordings      : 9 (3 with motion, 0 recovered), 0.07 GB, 0.1 h
Covering        : 2026-09-24 00:29 to 2026-09-24 00:52 (local time)
Motion events   : 3

24 September 2026 (local time)   ─ recording   █ motion   · nothing recorded
00:00 █····································· 24:00

Motion events (3):
  00:47:52  Motion detected  18.4 s, peak 11.2%  ->  2026-09-23/16-47-00Z.mp4, 2026-09-23/16-48-00Z.mp4
  ...

Recordings (9):
  00:47:00 - 00:48:00    60.0 s     9.4 MB  2026-09-23/16-47-00Z.mp4  motion
  00:48:00 - 00:49:00    60.0 s     9.4 MB  2026-09-23/16-48-00Z.mp4  motion
```

The walk that crossed a minute boundary should be listed with **two** files. Filenames stay in UTC (a day behind your local date around midnight), and the report shows local time.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `ABORTED, nothing changed` from the update script | Send me the message. |
| `Cannot open the database; recording continues without an index` | Send me the log lines that follow it. Recording is unaffected. |
| The report says `No database yet` | Run `python3 -m app.main` once from `~/surveillance`. |
| `sqlite3: command not found` | `sudo apt install -y sqlite3` (only needed for step 3) |

**Please send me:**
- the report from step 2 (showing the event that spans two files);
- the log lines from steps 4 and 5;
- the 60-second CPU figure from step 6.

Phase 8 is the local web dashboard: live status, the event timeline, and browsing recordings. It will listen only on the Pi itself until the login system (Phase 11) and private remote access (Phase 12) are in place.
