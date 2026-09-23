Phase 5 (storage management) is ready for you to run on the Pi. I tested it here with the same fake camera setup as before, but it hasn't run on your Pi yet.

## What it does

The storage manager checks every 30 seconds and again whenever a segment finishes.

- **Size limit.** While recordings take more than `max_storage_gb` (default 45 GB), it deletes the oldest finished segment. The segment currently being written (`.partial`) is never deleted, but its size does count towards the limit.
- **Free-space safety.** While free space is below `min_free_gb` (default 2 GB), it also deletes oldest first. If nothing is left to delete, it logs a "Storage low" warning and keeps recording.
- **Emergency pause.** If free space falls below half of `min_free_gb` (1 GB), writing pauses at the next keyframe. The camera keeps running, and writing resumes by itself once free space is back above 60 % (1.2 GB). The gap between the two thresholds stops it switching on and off repeatedly.
- **Optional age limit.** `max_age_days` deletes anything older (0 = off).
- **SSD guard for later.** `required_mount` (e.g. `/mnt/cctv`) makes the recorder refuse to record if that drive isn't mounted, instead of silently filling the SD card.
- **Crash and power-cut recovery.** A leftover `.partial` that has been untouched for more than 60 seconds is cut back to its last complete 2-second fragment, checked with `ffprobe`, and renamed `….recovered.mp4`. If nothing playable remains, it is deleted.
- **Logs.** Every deletion is logged, and a usage summary is logged every 10 minutes.

Everything is based on what's on disk; the SQLite index comes in Phase 7.

## What I tested

- **Deletion order:** oldest first, with the segment being written protected. Empty folders from earlier days were removed.
- **Recovery:** I cut a real fragmented recording at 70 % to simulate a power cut. Recovery removed the 61 KiB half-written tail, leaving 8.0 s that decodes with no errors. **Chrome showed the correct 8.00 s length and seeked within it.** A junk `.partial` was deleted.
- **End to end with the fake camera** (0.02 GB limit, 60 s segments): a crashed `.partial` was recovered before the camera started, deletions kept usage at or below the limit, and the file being written was never touched.
- **Free-space states:** with a 2 GB minimum, simulated free space of 10 → 1.5 → 0.9 → 1.1 → 1.3 → 2.5 GB gave `ok → low → critical (paused) → still paused → low (resumed) → ok`.
- **Pause and resume inside the recorder:** writing paused at the first keyframe after storage became critical, and resumed at the first keyframe after it recovered, in a new file.
- **Unmounted SSD path:** the recorder refused to start, retried with increasing waits, and created no folder on the SD card.
- **Invalid settings** were rejected: a recordings folder outside `required_mount`, text where a number belongs, and negative sizes.

## Files

| File | Status |
|---|---|
| `app/__init__.py` | Version 0.5.0 |
| `app/config.py` | New `storage` section |
| `app/storage_manager.py` | **New** |
| `app/recorder.py` | Pause and resume, refuses an unmounted drive, reports which file is being written |
| `app/main.py` | Starts the storage manager and runs recovery before the camera starts |
| `tools/phase5_storage_report.py` | **New.** Read-only report: usage, limits, bitrate, hours of history, what would be deleted |
| `tools/set_setting.py` | **New.** Change one setting safely (validated, atomic save) |

`fileutil.py`, `clock.py`, `camera.py`, `logging_setup.py` and `phase4_check_segments.py` are unchanged. Paste each block below from `~/surveillance`.

```bash
cat > ~/surveillance/app/__init__.py <<'EOF'
"""Raspberry Pi surveillance camera."""

__version__ = "0.5.0"
EOF
```

```bash
cat > ~/surveillance/app/config.py <<'EOF'
"""Typed, validated settings stored as JSON.

Every value is checked for type and range when loaded, so a typo or an out-of-range
value stops the service with a clear message instead of misbehaving later.
"""
from __future__ import annotations

import json
import re
from dataclasses import asdict, dataclass, field, fields
from pathlib import Path
from typing import Any

from app.fileutil import atomic_write_bytes

PROJECT_ROOT = Path(__file__).resolve().parent.parent
DEFAULT_CONFIG_PATH = PROJECT_ROOT / "config" / "settings.json"

ALLOWED_SEGMENT_SECONDS = (60, 120, 300, 600, 900, 1800, 3600)
LOG_LEVELS = ("DEBUG", "INFO", "WARNING", "ERROR")
LOG_FORMATS = ("text", "json")
RETENTION_MODES = ("oldest_first",)
MAX_ENCODER_PIXELS = 1920 * 1080
CAMERA_NAME_PATTERN = re.compile(r"^[\w][\w .,'()-]{0,39}$")


class ConfigError(ValueError):
    pass


@dataclass(frozen=True)
class CameraSettings:
    camera_num: int = 0
    name: str = "Camera 1"
    width: int = 1296
    height: int = 972
    lores_width: int = 640
    lores_height: int = 480
    framerate: float = 15.0
    bitrate: int = 2_500_000
    keyframe_seconds: float = 2.0
    rotate180: bool = False

    def validate(self) -> None:
        _check_range("camera.camera_num", self.camera_num, 0, 3)
        if not CAMERA_NAME_PATTERN.match(self.name):
            raise ConfigError("camera.name must be 1-40 letters, digits, spaces or . , ' ( ) - _")
        _check_range("camera.width", self.width, 64, 1920)
        _check_range("camera.height", self.height, 64, 1920)
        if self.width % 16 or self.height % 2:
            raise ConfigError("camera.width must be a multiple of 16 and camera.height must be even")
        if self.width * self.height > MAX_ENCODER_PIXELS:
            raise ConfigError("camera.width x camera.height exceeds the 1920x1080 hardware encoder limit")
        _check_range("camera.lores_width", self.lores_width, 160, self.width)
        _check_range("camera.lores_height", self.lores_height, 120, self.height)
        if self.lores_width % 16 or self.lores_height % 2:
            raise ConfigError("camera.lores_width must be a multiple of 16 and lores_height even")
        _check_range("camera.framerate", self.framerate, 1.0, 30.0)
        _check_range("camera.bitrate", self.bitrate, 250_000, 10_000_000)
        _check_range("camera.keyframe_seconds", self.keyframe_seconds, 0.5, 10.0)

    @property
    def keyframe_interval_frames(self) -> int:
        return max(1, round(self.keyframe_seconds * self.framerate))


@dataclass(frozen=True)
class RecordingSettings:
    segment_seconds: int = 300

    def validate(self) -> None:
        if self.segment_seconds not in ALLOWED_SEGMENT_SECONDS:
            allowed = ", ".join(str(value) for value in ALLOWED_SEGMENT_SECONDS)
            raise ConfigError(f"recording.segment_seconds must be one of: {allowed}")


@dataclass(frozen=True)
class StorageSettings:
    max_storage_gb: float = 45.0
    min_free_gb: float = 2.0
    max_age_days: int = 0
    retention: str = "oldest_first"
    required_mount: str = ""

    def validate(self) -> None:
        _check_range("storage.max_storage_gb", self.max_storage_gb, 0.01, 100_000)
        _check_range("storage.min_free_gb", self.min_free_gb, 0.1, 10_000)
        _check_range("storage.max_age_days", self.max_age_days, 0, 3650)
        if self.retention not in RETENTION_MODES:
            raise ConfigError(f"storage.retention must be one of: {', '.join(RETENTION_MODES)}")
        if self.required_mount and not Path(self.required_mount).is_absolute():
            raise ConfigError("storage.required_mount must be empty or an absolute path such as /mnt/cctv")


@dataclass(frozen=True)
class PathSettings:
    recordings_dir: str = "recordings"
    log_dir: str = "logs"

    def validate(self) -> None:
        for name in ("recordings_dir", "log_dir"):
            value = getattr(self, name)
            if not value.strip() or "\x00" in value:
                raise ConfigError(f"paths.{name} must be a non-empty path")


@dataclass(frozen=True)
class LoggingSettings:
    level: str = "INFO"
    format: str = "text"
    console: bool = True
    max_bytes: int = 5_000_000
    backup_count: int = 5

    def validate(self) -> None:
        if self.level not in LOG_LEVELS:
            raise ConfigError(f"logging.level must be one of: {', '.join(LOG_LEVELS)}")
        if self.format not in LOG_FORMATS:
            raise ConfigError(f"logging.format must be one of: {', '.join(LOG_FORMATS)}")
        _check_range("logging.max_bytes", self.max_bytes, 100_000, 50_000_000)
        _check_range("logging.backup_count", self.backup_count, 1, 20)


@dataclass(frozen=True)
class Settings:
    camera: CameraSettings = field(default_factory=CameraSettings)
    recording: RecordingSettings = field(default_factory=RecordingSettings)
    storage: StorageSettings = field(default_factory=StorageSettings)
    paths: PathSettings = field(default_factory=PathSettings)
    logging: LoggingSettings = field(default_factory=LoggingSettings)

    def validate(self) -> None:
        for section in fields(self):
            getattr(self, section.name).validate()
        mount = self.storage.required_mount
        if mount and not self.recordings_dir.is_relative_to(Path(mount)):
            raise ConfigError(f"paths.recordings_dir ({self.recordings_dir}) must be inside "
                              f"storage.required_mount ({mount})")

    @property
    def recordings_dir(self) -> Path:
        return resolve_path(self.paths.recordings_dir)

    @property
    def log_dir(self) -> Path:
        return resolve_path(self.paths.log_dir)


def resolve_path(value: str) -> Path:
    """Relative paths are relative to the project directory, so the app works from any cwd."""
    path = Path(value).expanduser()
    return path if path.is_absolute() else PROJECT_ROOT / path


def _check_range(name: str, value: float, low: float, high: float) -> None:
    if not low <= value <= high:
        raise ConfigError(f"{name} must be between {low:g} and {high:g} (got {value!r})")


def _coerce(value: Any, expected: type, name: str) -> Any:
    if expected is bool:
        if isinstance(value, bool):
            return value
    elif expected is int:
        if isinstance(value, int) and not isinstance(value, bool):
            return value
    elif expected is float:
        if isinstance(value, (int, float)) and not isinstance(value, bool):
            return float(value)
    elif expected is str:
        if isinstance(value, str):
            return value
    raise ConfigError(f"{name} must be of type {expected.__name__} (got {value!r})")


def _build_section(cls: type, data: Any, section: str) -> Any:
    if not isinstance(data, dict):
        raise ConfigError(f"'{section}' must be a JSON object")
    known = {f.name for f in fields(cls)}
    unknown = sorted(set(data) - known)
    if unknown:
        raise ConfigError(f"unknown setting(s) in '{section}': {', '.join(unknown)}")
    defaults = cls()
    values = {}
    for f in fields(cls):
        default = getattr(defaults, f.name)
        values[f.name] = (_coerce(data[f.name], type(default), f"{section}.{f.name}")
                          if f.name in data else default)
    return cls(**values)


def settings_from_dict(data: Any) -> Settings:
    if not isinstance(data, dict):
        raise ConfigError("settings file must contain a JSON object")
    section_types = {f.name: type(getattr(Settings(), f.name)) for f in fields(Settings)}
    unknown = sorted(set(data) - set(section_types))
    if unknown:
        raise ConfigError(f"unknown section(s): {', '.join(unknown)}")
    settings = Settings(**{name: _build_section(cls, data.get(name, {}), name)
                           for name, cls in section_types.items()})
    settings.validate()
    return settings


def load_settings(path: Path = DEFAULT_CONFIG_PATH, create_if_missing: bool = True) -> tuple[Settings, bool]:
    """Return (settings, created). A missing file is created with defaults if allowed."""
    if not path.exists():
        if not create_if_missing:
            raise ConfigError(f"{path} does not exist")
        settings = Settings()
        save_settings(settings, path)
        return settings, True
    try:
        data = json.loads(path.read_text(encoding="utf-8"))
    except json.JSONDecodeError as exc:
        raise ConfigError(f"{path}: invalid JSON at line {exc.lineno}, column {exc.colno}: {exc.msg}") from None
    except OSError as exc:
        raise ConfigError(f"cannot read {path}: {exc.strerror}") from None
    return settings_from_dict(data), False


def save_settings(settings: Settings, path: Path = DEFAULT_CONFIG_PATH) -> None:
    settings.validate()
    text = json.dumps(asdict(settings), indent=2) + "\n"
    atomic_write_bytes(path, text.encode("utf-8"))
EOF
```

```bash
cat > ~/surveillance/app/storage_manager.py <<'EOF'
"""Storage quota, free-space safety and crash recovery for recording segments.

Rules:
  * Only completed segments (*.mp4) are ever deleted, oldest first. The segment being
    written (*.mp4.partial) is never touched.
  * Delete while usage > max_storage_gb, or free space < min_free_gb, or a segment is older
    than max_age_days (if set).
  * If free space falls below half of min_free_gb with nothing left to delete, writing pauses
    (the camera keeps running) and resumes automatically once space is available again.
  * Unfinished .partial files left by a crash or power cut are trimmed to their last complete
    fragment and renamed *.recovered.mp4, or deleted if nothing playable remains.
"""
from __future__ import annotations

import logging
import os
import shutil
import struct
import subprocess
import threading
import time
from dataclasses import dataclass
from datetime import datetime, timezone
from pathlib import Path
from typing import Callable

from app.config import Settings
from app.fileutil import fsync_directory

log = logging.getLogger("StorageManager")

GB = 1_000_000_000
CHECK_INTERVAL_S = 30.0
USAGE_LOG_INTERVAL_S = 600.0
PARTIAL_STALE_S = 60.0
MIN_PLAYABLE_S = 1.0
CRITICAL_FRACTION = 0.5
RESUME_FRACTION = 0.6
MAX_INDIVIDUAL_DELETE_LOGS = 5

SEGMENT_SUFFIX = ".mp4"
PARTIAL_SUFFIX = ".mp4.partial"
RECOVERED_SUFFIX = ".recovered.mp4"


@dataclass(frozen=True)
class SegmentFile:
    path: Path
    rel_path: str
    size: int          # bytes actually allocated on disk
    start: float       # UTC epoch seconds, from the file name (mtime as fallback)
    mtime: float
    partial: bool


@dataclass(frozen=True)
class StorageStatus:
    state: str         # ok | low | critical | unavailable
    message: str
    used_bytes: int
    free_bytes: int
    total_bytes: int
    max_bytes: int
    min_free_bytes: int
    segment_count: int
    oldest_start: float | None
    newest_start: float | None


def parse_segment_start(day: str, name: str) -> float | None:
    """'2026-09-23', '14-05-00Z.mp4' -> UTC epoch seconds."""
    stem = name.split("Z", 1)[0]
    try:
        return datetime.strptime(f"{day} {stem}", "%Y-%m-%d %H-%M-%S").replace(
            tzinfo=timezone.utc).timestamp()
    except ValueError:
        return None


def scan_segments(root: Path) -> list[SegmentFile]:
    """All segment and partial files under root, oldest first."""
    found: list[SegmentFile] = []
    try:
        days = [entry for entry in os.scandir(root) if entry.is_dir(follow_symlinks=False)]
    except FileNotFoundError:
        return found
    for day in days:
        try:
            entries = list(os.scandir(day.path))
        except OSError as exc:
            log.error("Cannot read %s: %s", day.path, exc.strerror)
            continue
        for entry in entries:
            name = entry.name
            partial = name.endswith(PARTIAL_SUFFIX)
            if not (partial or name.endswith(SEGMENT_SUFFIX)) or name.startswith("."):
                continue
            try:
                st = entry.stat(follow_symlinks=False)
            except FileNotFoundError:
                continue
            start = parse_segment_start(day.name, name)
            found.append(SegmentFile(path=Path(entry.path), rel_path=f"{day.name}/{name}",
                                     size=st.st_blocks * 512, start=start if start is not None else st.st_mtime,
                                     mtime=st.st_mtime, partial=partial))
    found.sort(key=lambda f: (f.start, f.rel_path))
    return found


def plan_deletions(files: list[SegmentFile], *, now: float, max_bytes: int, min_free_bytes: int,
                   free_bytes: int, max_age_days: int, protected: set[Path]) -> list[tuple[SegmentFile, str]]:
    """Pure decision function: which completed segments to delete (oldest first) and why."""
    usage = sum(f.size for f in files)
    cutoff = now - max_age_days * 86400 if max_age_days else None
    plan = []
    for f in files:
        if f.partial or f.path in protected:
            continue
        if cutoff is not None and f.start < cutoff:
            reason = f"older than {max_age_days} day(s)"
        elif usage > max_bytes:
            reason = f"over the {max_bytes / GB:g} GB limit"
        elif free_bytes < min_free_bytes:
            reason = f"free space below {min_free_bytes / GB:g} GB"
        else:
            break
        plan.append((f, reason))
        usage -= f.size
        free_bytes += f.size
    return plan


def trim_to_complete_fragments(path: Path) -> tuple[int, int]:
    """Cut a fragmented MP4 after its last complete fragment. Returns (old_size, new_size).

    A power cut leaves a half-written moof/mdat box at the end; removing it makes the
    file well-formed so browsers play it to the end instead of stalling.
    """
    with open(path, "r+b") as handle:
        size = os.fstat(handle.fileno()).st_size
        pos = last_good = 0
        pending_moof = False
        while pos + 8 <= size:
            handle.seek(pos)
            header = handle.read(16)
            box_size, box_type = struct.unpack(">I4s", header[:8])
            if box_size == 1:
                if len(header) < 16:
                    break
                box_size = struct.unpack(">Q", header[8:16])[0]
            elif box_size == 0:
                box_size = size - pos
            if box_size < 8 or pos + box_size > size:
                break
            pos += box_size
            if box_type == b"moof":
                pending_moof = True
            elif box_type == b"mdat":
                pending_moof = False
                last_good = pos
            elif not pending_moof:
                last_good = pos
        if last_good < size:
            handle.truncate(last_good)
            handle.flush()
            os.fsync(handle.fileno())
    return size, last_good


def playable_seconds(path: Path) -> float | None:
    """Seconds of decodable video according to ffprobe, or None if ffprobe is unavailable."""
    try:
        result = subprocess.run(
            ["ffprobe", "-v", "error", "-select_streams", "v:0",
             "-show_entries", "packet=pts_time", "-of", "csv=p=0", str(path)],
            capture_output=True, text=True, timeout=120, check=False)
    except (OSError, subprocess.TimeoutExpired):
        return None
    times = []
    for line in result.stdout.splitlines():
        try:
            times.append(float(line.strip().rstrip(",")))
        except ValueError:
            continue
    return max(times) - min(times) if len(times) > 1 else 0.0


class StorageManager:
    def __init__(self, settings: Settings, current_segment: Callable[[], Path | None]) -> None:
        s = settings.storage
        self.root = settings.recordings_dir
        self._required_mount = Path(s.required_mount) if s.required_mount else None
        self._max_bytes = int(s.max_storage_gb * GB)
        self._min_free_bytes = int(s.min_free_gb * GB)
        self._max_age_days = s.max_age_days
        self._current_segment = current_segment
        self._lock = threading.Lock()
        self._wake = threading.Event()
        self._stop = threading.Event()
        self._thread = threading.Thread(target=self._run, name="storage-manager", daemon=True)
        self._writable = True
        self._last_usage_log = 0.0
        self._warned_capacity = False
        self._ffprobe_warned = False
        self.status: StorageStatus | None = None

    def location_error(self) -> str | None:
        """Why recordings must not be written right now (e.g. the SSD is not mounted), or None."""
        if self._required_mount is not None and not os.path.ismount(self._required_mount):
            return f"{self._required_mount} is not mounted; refusing to record onto the SD card"
        return None

    def can_write(self) -> bool:
        return self._writable

    def trigger(self) -> None:
        self._wake.set()

    def start(self) -> None:
        self._thread.start()

    def stop(self) -> None:
        self._stop.set()
        self._wake.set()
        if self._thread.is_alive():
            self._thread.join(timeout=30)

    def _run(self) -> None:
        while not self._stop.is_set():
            self._wake.wait(CHECK_INTERVAL_S)
            self._wake.clear()
            if self._stop.is_set():
                return
            try:
                self.run_cycle()
            except Exception:  # noqa: BLE001 - never let the storage thread die
                log.exception("Storage check failed")

    def run_cycle(self) -> StorageStatus:
        with self._lock:
            problem = self.location_error()
            if problem:
                return self._set_status("unavailable", problem, [], 0, 0)
            self.root.mkdir(parents=True, exist_ok=True)
            current = self._current_segment()
            files = scan_segments(self.root)
            if self._recover_partials(files, current):
                files = scan_segments(self.root)

            disk = shutil.disk_usage(self.root)
            protected = {current} if current else set()
            plan = plan_deletions(files, now=time.time(), max_bytes=self._max_bytes,
                                  min_free_bytes=self._min_free_bytes, free_bytes=disk.free,
                                  max_age_days=self._max_age_days, protected=protected)
            if plan:
                self._delete(plan)
                files = scan_segments(self.root)
                disk = shutil.disk_usage(self.root)
            return self._evaluate(files, disk)

    def _evaluate(self, files: list[SegmentFile], disk) -> StorageStatus:
        used = sum(f.size for f in files)
        if not self._warned_capacity and self._max_bytes > used + disk.free - self._min_free_bytes:
            self._warned_capacity = True
            log.warning("Maximum storage %.1f GB does not fit on this disk while keeping %.1f GB free "
                        "(room for about %.1f GB); the free-space limit will apply first",
                        self._max_bytes / GB, self._min_free_bytes / GB,
                        max(0, used + disk.free - self._min_free_bytes) / GB)

        critical_below = self._min_free_bytes * CRITICAL_FRACTION
        resume_above = self._min_free_bytes * RESUME_FRACTION
        if disk.free < critical_below or (not self._writable and disk.free < resume_above):
            state = "critical"
            message = (f"only {disk.free / GB:.2f} GB free and nothing left to delete; "
                       f"recording is PAUSED until more space is available")
        elif disk.free < self._min_free_bytes:
            state = "low"
            message = (f"only {disk.free / GB:.2f} GB free (minimum {self._min_free_bytes / GB:g} GB); "
                       f"something other than recordings is filling the disk")
        elif used > self._max_bytes:
            state = "low"
            message = f"recordings use {used / GB:.2f} GB, above the {self._max_bytes / GB:g} GB limit"
        else:
            state = "ok"
            message = "ok"
        return self._set_status(state, message, files, used, disk.free, disk.total)

    def _set_status(self, state: str, message: str, files: list[SegmentFile], used: int, free: int,
                    total: int = 0) -> StorageStatus:
        previous = self.status.state if self.status else None
        if state != previous:
            if state == "ok":
                if previous is not None:
                    log.info("Storage back to normal")
            elif state == "low":
                log.warning("Storage low: %s", message)
            else:
                log.error("Storage %s: %s", state, message)
        self._writable = state in ("ok", "low")

        complete = [f for f in files if not f.partial]
        self.status = StorageStatus(
            state=state, message=message, used_bytes=used, free_bytes=free, total_bytes=total,
            max_bytes=self._max_bytes, min_free_bytes=self._min_free_bytes, segment_count=len(complete),
            oldest_start=complete[0].start if complete else None,
            newest_start=complete[-1].start if complete else None)

        now = time.monotonic()
        if state != "unavailable" and now - self._last_usage_log >= USAGE_LOG_INTERVAL_S:
            self._last_usage_log = now
            oldest = (datetime.fromtimestamp(self.status.oldest_start, timezone.utc).strftime("%Y-%m-%d %H:%M UTC")
                      if self.status.oldest_start else "none")
            log.info("Storage usage %.2f GB / %g GB, %.1f GB free, %d segments, oldest %s",
                     used / GB, self._max_bytes / GB, free / GB, len(complete), oldest)
        return self.status

    def _delete(self, plan: list[tuple[SegmentFile, str]]) -> None:
        freed = 0
        deleted = 0
        touched_days: set[Path] = set()
        for f, reason in plan:
            try:
                f.path.unlink()
            except FileNotFoundError:
                continue
            except OSError as exc:
                log.error("Cannot delete %s: %s", f.rel_path, exc.strerror)
                continue
            deleted += 1
            freed += f.size
            touched_days.add(f.path.parent)
            if deleted <= MAX_INDIVIDUAL_DELETE_LOGS:
                log.info("Deleted %s (%.1f MB): %s", f.rel_path, f.size / 1e6, reason)
        if deleted > MAX_INDIVIDUAL_DELETE_LOGS:
            log.info("Deleted %d segments in total, freeing %.1f GB", deleted, freed / GB)
        today = datetime.now(timezone.utc).strftime("%Y-%m-%d")
        for day in touched_days:
            if day.name < today:
                try:
                    day.rmdir()
                except OSError:
                    pass
        if touched_days:
            fsync_directory(self.root)

    def _recover_partials(self, files: list[SegmentFile], current: Path | None) -> bool:
        changed = False
        now = time.time()
        for f in files:
            if not f.partial or f.path == current or now - f.mtime < PARTIAL_STALE_S:
                continue
            changed |= self._recover(f)
        return changed

    def _recover(self, f: SegmentFile) -> bool:
        try:
            old_size, new_size = trim_to_complete_fragments(f.path)
        except OSError as exc:
            log.error("Cannot read unfinished segment %s: %s", f.rel_path, exc.strerror)
            return False
        seconds = playable_seconds(f.path) if new_size else 0.0
        if seconds is None:
            if not self._ffprobe_warned:
                self._ffprobe_warned = True
                log.error("ffprobe is not available; cannot check unfinished segments (sudo apt install ffmpeg)")
            return False
        if seconds < MIN_PLAYABLE_S:
            f.path.unlink(missing_ok=True)
            log.warning("Deleted unfinished segment %s: nothing playable (%.1f MB)", f.rel_path, old_size / 1e6)
            return True
        recovered = f.path.with_name(f.path.name[: -len(PARTIAL_SUFFIX)] + RECOVERED_SUFFIX)
        os.replace(f.path, recovered)
        fsync_directory(recovered.parent)
        trimmed = old_size - new_size
        log.warning("Recovered unfinished segment %s -> %s: %.1f s playable%s", f.rel_path, recovered.name,
                    seconds, f", removed {trimmed / 1024:.0f} KiB of incomplete data" if trimmed else "")
        return True
EOF
```

```bash
cat > ~/surveillance/app/recorder.py <<'EOF'
"""Continuous recording into wall-clock-aligned, keyframe-split fragmented MP4 segments.

Files are named in UTC:  <recordings_dir>/YYYY-MM-DD/HH-MM-SSZ.mp4
While a segment is being written it is called  HH-MM-SSZ.mp4.partial ; it is renamed to .mp4
only after it has been closed and flushed to disk, so a ".mp4" file is always complete.
"""
from __future__ import annotations

import logging
import math
import os
import queue
import threading
import time
from dataclasses import dataclass
from datetime import datetime, timezone
from pathlib import Path
from typing import Callable

# PyavOutput imports av lazily on first use, which would happen inside the encoder thread when the
# first segment opens; on a Pi 4 that import takes long enough to drop frames.
import av  # noqa: F401
from picamera2.encoders import H264Encoder
from picamera2.outputs import Output, PyavOutput

from app.camera import Camera
from app.clock import ClockMonitor
from app.config import Settings
from app.fileutil import fsync_directory, fsync_file

log = logging.getLogger("Recorder")

FRAGMENTED_MOVFLAGS = "frag_keyframe+empty_moov+default_base_moof"
PARTIAL_SUFFIX = ".partial"
NO_FRAMES_TIMEOUT_S = 10.0
STARTUP_TIMEOUT_S = 15.0
OPEN_RETRY_S = 10.0


@dataclass(frozen=True)
class Segment:
    """A completed recording segment."""
    path: Path
    rel_path: str
    start_utc: datetime
    end_utc: datetime
    duration_s: float
    size_bytes: int
    frames: int
    clock_synced: bool | None


class _OpenSegment:
    def __init__(self, final_path: Path, output: PyavOutput, start_wall: float, first_ts: int,
                 boundary: float, clock_synced: bool | None) -> None:
        self.final_path = final_path
        self.partial_path = final_path.with_name(final_path.name + PARTIAL_SUFFIX)
        self.output = output
        self.start_wall = start_wall
        self.first_ts = first_ts
        self.last_ts = first_ts
        self.boundary = boundary
        self.clock_synced = clock_synced
        self.frames = 0
        self.error: str | None = None


class SegmentingOutput(Output):
    """Picamera2 output that starts a new file at the first keyframe after each segment boundary.

    outputframe() runs in Picamera2's encoder thread, so it never waits for the disk: closed
    segments are handed to `on_closed`, which fsyncs and renames them in another thread.
    """

    def __init__(self, recordings_dir: Path, segment_seconds: int, framerate: float,
                 keyframe_seconds: float, clock: ClockMonitor,
                 on_closed: Callable[[_OpenSegment], None], can_write: Callable[[], bool]) -> None:
        super().__init__()
        self._can_write = can_write
        self.paused = False
        self.needs_add_stream = True
        self._streams: list[tuple] = []
        self._dir = recordings_dir
        self._segment_seconds = segment_seconds
        self._frame_interval_us = 1_000_000 / framerate
        # A clock jump backwards must not produce one enormous segment.
        self._max_duration_us = (segment_seconds + 3 * keyframe_seconds) * 1_000_000
        self._align_tolerance_s = keyframe_seconds + 3.0
        self._clock = clock
        self._on_closed = on_closed
        self._lock = threading.Lock()
        self._current: _OpenSegment | None = None
        self._retry_open_at = 0.0
        self.last_frame_monotonic: float | None = None
        self.write_error: str | None = None
        self.write_error_since: float | None = None

    @property
    def current_rel_path(self) -> str | None:
        seg = self._current
        return str(seg.final_path.relative_to(self._dir)) if seg else None

    @property
    def current_partial_path(self) -> Path | None:
        seg = self._current
        return seg.partial_path if seg else None

    def _add_stream(self, encoder_stream, codec_name, **kwargs) -> None:
        self._streams.append((encoder_stream, codec_name, kwargs))

    def stop(self) -> None:
        with self._lock:
            super().stop()
            self._close_current()

    def outputframe(self, frame, keyframe=True, timestamp=None, packet=None, audio=False) -> None:
        if audio:
            return
        with self._lock:
            if not self.recording:
                return
            self.last_frame_monotonic = time.monotonic()
            if timestamp is None:
                timestamp = int(self.last_frame_monotonic * 1_000_000)
            seg = self._current
            if keyframe and not self._can_write():
                if not self.paused:
                    log.error("Recording paused: storage is critically full")
                    self.paused = True
                self._close_current()
                return
            if keyframe and self.paused:
                log.info("Recording resumed: storage space is available again")
                self.paused = False
            if keyframe and self._should_rotate(seg, timestamp):
                self._close_current()
                seg = self._open(timestamp)
            if seg is None:
                return
            seg.output.outputframe(frame, keyframe, timestamp - seg.first_ts, packet, audio)
            seg.last_ts = timestamp
            seg.frames += 1

    def _should_rotate(self, seg: _OpenSegment | None, timestamp: int) -> bool:
        if seg is None:
            return time.monotonic() >= self._retry_open_at
        return (seg.error is not None
                or time.time() >= seg.boundary
                or timestamp - seg.first_ts >= self._max_duration_us)

    def _open(self, timestamp: int) -> _OpenSegment | None:
        now = time.time()
        slot_start = math.floor(now / self._segment_seconds) * self._segment_seconds
        # Name aligned segments after their boundary (e.g. 13-05-00Z) even though the first
        # keyframe arrives up to one keyframe interval later; the true start is in the metadata.
        nominal = slot_start if now - slot_start <= self._align_tolerance_s else now
        final_path = self._unique_path(datetime.fromtimestamp(nominal, timezone.utc))
        seg = None
        try:
            final_path.parent.mkdir(parents=True, exist_ok=True)
            output = PyavOutput(str(final_path) + PARTIAL_SUFFIX, format="mp4",
                                options={"movflags": FRAGMENTED_MOVFLAGS})
            seg = _OpenSegment(final_path, output, now, timestamp, slot_start + self._segment_seconds,
                               self._clock.synchronized)
            output.error_callback = lambda exc, s=seg: setattr(s, "error", f"{type(exc).__name__}: {exc}")
            output.start()
            for encoder_stream, codec_name, kwargs in self._streams:
                output._add_stream(encoder_stream, codec_name, **kwargs)
        except Exception as exc:  # noqa: BLE001 - must never kill the encoder thread
            if seg is not None:
                try:
                    seg.output.stop()
                except Exception:  # noqa: BLE001
                    pass
            self._retry_open_at = time.monotonic() + OPEN_RETRY_S
            message = f"{type(exc).__name__}: {exc}"
            if self.write_error is None:
                self.write_error_since = time.monotonic()
                log.error("Cannot create segment %s: %s (retrying every %.0f s)",
                          final_path.name, message, OPEN_RETRY_S)
            self.write_error = message
            return None
        if self.write_error is not None:
            log.info("Segment writing recovered after error: %s", self.write_error)
        self.write_error = None
        self.write_error_since = None
        self._current = seg
        log.debug("Opened segment %s", final_path.relative_to(self._dir))
        return seg

    def _unique_path(self, start: datetime) -> Path:
        day_dir = self._dir / start.strftime("%Y-%m-%d")
        base = start.strftime("%H-%M-%SZ")
        candidate = day_dir / f"{base}.mp4"
        counter = 1
        while candidate.exists() or candidate.with_name(candidate.name + PARTIAL_SUFFIX).exists():
            candidate = day_dir / f"{base}-{counter}.mp4"
            counter += 1
        return candidate

    def _close_current(self) -> None:
        seg, self._current = self._current, None
        if seg is None:
            return
        try:
            seg.output.stop()
        except Exception as exc:  # noqa: BLE001
            seg.error = seg.error or f"{type(exc).__name__}: {exc}"
        self._on_closed(seg)

    def segment_metadata(self, seg: _OpenSegment, size_bytes: int) -> Segment:
        duration_s = ((seg.last_ts - seg.first_ts) + self._frame_interval_us) / 1_000_000
        start = datetime.fromtimestamp(seg.start_wall, timezone.utc)
        end = datetime.fromtimestamp(seg.start_wall + duration_s, timezone.utc)
        return Segment(path=seg.final_path, rel_path=str(seg.final_path.relative_to(self._dir)),
                       start_utc=start, end_utc=end, duration_s=round(duration_s, 3),
                       size_bytes=size_bytes, frames=seg.frames, clock_synced=seg.clock_synced)


class Recorder:
    """Owns the camera, the hardware H.264 encoder and the segment files."""

    def __init__(self, settings: Settings, clock: ClockMonitor,
                 can_write: Callable[[], bool] = lambda: True,
                 location_error: Callable[[], str | None] = lambda: None) -> None:
        self.settings = settings
        self.recordings_dir = settings.recordings_dir
        self._clock = clock
        self._can_write = can_write
        self._location_error = location_error
        self._camera: Camera | None = None
        self._output: SegmentingOutput | None = None
        self._recording = False
        self._started_monotonic = 0.0
        self._finalize_queue: queue.Queue[_OpenSegment | None] = queue.Queue()
        self._finalizer = threading.Thread(target=self._finalize_loop, name="segment-finalizer", daemon=True)
        self._boundary_stop = threading.Event()
        self._boundary_thread: threading.Thread | None = None
        self._listeners: list[Callable[[Segment], None]] = []
        self.last_segment: Segment | None = None

    def add_listener(self, callback: Callable[[Segment], None]) -> None:
        self._listeners.append(callback)

    @property
    def current_segment(self) -> str | None:
        return self._output.current_rel_path if self._output else None

    @property
    def current_partial_path(self) -> Path | None:
        return self._output.current_partial_path if self._output else None

    def start(self) -> None:
        cam = self.settings.camera
        problem = self._location_error()
        if problem:
            raise OSError(problem)
        self.recordings_dir.mkdir(parents=True, exist_ok=True)
        if not os.access(self.recordings_dir, os.W_OK):
            raise PermissionError(f"recordings directory is not writable: {self.recordings_dir}")
        self._finalizer.start()

        self._camera = Camera(cam)
        self._camera.open()
        encoder = H264Encoder(bitrate=cam.bitrate, iperiod=cam.keyframe_interval_frames,
                              framerate=cam.framerate, profile="high", repeat=True)
        self._output = SegmentingOutput(self.recordings_dir, self.settings.recording.segment_seconds,
                                        cam.framerate, cam.keyframe_seconds, self._clock,
                                        self._finalize_queue.put, self._can_write)
        self._camera.picam2.start_recording(encoder, self._output)
        self._recording = True
        self._started_monotonic = time.monotonic()
        self._boundary_thread = threading.Thread(target=self._boundary_loop, args=(encoder,),
                                                 name="segment-boundaries", daemon=True)
        self._boundary_thread.start()
        log.info("Started recording to %s (%d s segments, %.2f Mbit/s, keyframe every %g s)",
                 self.recordings_dir, self.settings.recording.segment_seconds, cam.bitrate / 1e6,
                 cam.keyframe_seconds)

    def stop(self) -> None:
        was_recording = self._recording
        self._boundary_stop.set()
        if self._boundary_thread is not None and self._boundary_thread.is_alive():
            self._boundary_thread.join(timeout=5)
        if self._camera is not None and self._camera.picam2 is not None:
            if self._recording:
                try:
                    self._camera.picam2.stop_recording()
                except Exception as exc:  # noqa: BLE001
                    log.error("Error while stopping the encoder: %s", exc)
                    if self._output is not None:
                        self._output.stop()
            self._camera.close()
        self._camera = None
        self._recording = False
        if self._finalizer.is_alive():
            self._finalize_queue.put(None)
            self._finalizer.join(timeout=60)
        if was_recording:
            log.info("Stopped recording")

    def health(self) -> str | None:
        """Return a description of the problem, or None if frames are flowing and being written."""
        if not self._recording or self._output is None:
            return "not recording"
        now = time.monotonic()
        last = self._output.last_frame_monotonic
        if last is None:
            if now - self._started_monotonic > STARTUP_TIMEOUT_S:
                return f"no frames received {STARTUP_TIMEOUT_S:.0f} s after start"
        elif now - last > NO_FRAMES_TIMEOUT_S:
            return f"no frames for {now - last:.0f} s"
        return None

    @property
    def write_error(self) -> str | None:
        return self._output.write_error if self._output else None

    def _boundary_loop(self, encoder: H264Encoder) -> None:
        """Request a keyframe at each boundary so segments start on time, not up to 2 s late."""
        segment_seconds = self.settings.recording.segment_seconds
        while True:
            now = time.time()
            next_boundary = (math.floor(now / segment_seconds) + 1) * segment_seconds
            if self._boundary_stop.wait(next_boundary - now):
                return
            try:
                encoder.force_key_frame()
            except Exception as exc:  # noqa: BLE001 - regular keyframes still split the segment
                log.debug("Could not force a keyframe: %s", exc)

    def _finalize_loop(self) -> None:
        while True:
            seg = self._finalize_queue.get()
            if seg is None:
                return
            try:
                self._finalize(seg)
            except Exception:  # noqa: BLE001 - one bad segment must not stop finalisation
                log.exception("Failed to finalise segment %s", seg.partial_path)

    def _finalize(self, seg: _OpenSegment) -> None:
        partial = seg.partial_path
        if not partial.exists():
            log.error("Segment file vanished before it was finalised: %s", partial)
            return
        if seg.frames == 0:
            partial.unlink(missing_ok=True)
            return
        fsync_file(partial)
        if seg.error:
            log.error("Segment %s had a write error (%s); left as .partial for recovery",
                      partial.name, seg.error)
            return
        os.replace(partial, seg.final_path)
        fsync_directory(seg.final_path.parent)
        segment = self._output.segment_metadata(seg, seg.final_path.stat().st_size)
        self.last_segment = segment
        log.info("Segment completed: %s start=%s duration=%.1fs frames=%d size=%.1fMB%s",
                 segment.rel_path, segment.start_utc.isoformat(timespec="milliseconds"),
                 segment.duration_s, segment.frames, segment.size_bytes / 1e6,
                 {True: "", False: " (clock NOT synchronised)"}.get(segment.clock_synced,
                                                                     " (clock sync unknown)"))
        for callback in self._listeners:
            try:
                callback(segment)
            except Exception:  # noqa: BLE001
                log.exception("Segment listener failed")
EOF
```

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
from app.logging_setup import setup_logging  # noqa: E402
from app.recorder import Recorder  # noqa: E402
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


def run(settings: Settings, stop: threading.Event) -> None:
    clock = ClockMonitor()
    clock.start()
    active: list[Recorder] = []
    storage = StorageManager(settings, lambda: active[0].current_partial_path if active else None)
    # Recover unfinished segments and free space before the camera starts writing.
    try:
        storage.run_cycle()
    except Exception:  # noqa: BLE001 - the periodic check retries; recording matters more
        log.exception("Initial storage check failed")
    storage.start()
    backoff = BACKOFF_INITIAL_S
    try:
        while not stop.is_set():
            recorder = Recorder(settings, clock, storage.can_write, storage.location_error)
            recorder.add_listener(lambda _segment: storage.trigger())
            active[:] = [recorder]
            try:
                recorder.start()
            except (CameraError, OSError) as exc:
                log.error("Could not start recording: %s. Retrying in %d s", exc, backoff)
                recorder.stop()
                stop.wait(backoff)
                backoff = min(backoff * 2, BACKOFF_MAX_S)
                continue

            started = time.monotonic()
            while not stop.wait(1.0):
                problem = recorder.health()
                if problem:
                    log.error("Recording unhealthy: %s. Restarting the camera pipeline", problem)
                    break
                if time.monotonic() - started > HEALTHY_RESET_S:
                    backoff = BACKOFF_INITIAL_S
            recorder.stop()
            if not stop.is_set():
                stop.wait(backoff)
                backoff = min(backoff * 2, BACKOFF_MAX_S)
    finally:
        active.clear()
        storage.stop()
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

```bash
cat > ~/surveillance/tools/phase5_storage_report.py <<'EOF'
#!/usr/bin/env python3
"""Phase 5: read-only storage report. Changes nothing on disk.

    python3 ~/surveillance/tools/phase5_storage_report.py [--max-gb N] [--min-free-gb N]

Shows where recordings are stored, how much space they use, the configured limits, the
average bitrate, how many hours of history the limit holds, and which segments the storage
manager would delete right now (or with the limits given on the command line).
"""
from __future__ import annotations

import argparse
import os
import shutil
import sys
import time
from datetime import datetime, timezone
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent.parent))

from app.config import DEFAULT_CONFIG_PATH, ConfigError, load_settings  # noqa: E402
from app.storage_manager import (GB, RECOVERED_SUFFIX, plan_deletions,  # noqa: E402
                                 scan_segments)

BITRATE_SAMPLE_SEGMENTS = 12


def mount_point(path: Path) -> Path:
    path = path.resolve()
    while not os.path.ismount(path):
        path = path.parent
    return path


def mount_device(mount: Path) -> str:
    try:
        for line in Path("/proc/mounts").read_text().splitlines():
            device, where, fstype, *_ = line.split()
            if where == str(mount):
                return f"{device} ({fstype})"
    except OSError:
        pass
    return "unknown device"


def fmt_time(epoch: float | None) -> str:
    if epoch is None:
        return "none"
    utc = datetime.fromtimestamp(epoch, timezone.utc)
    local = utc.astimezone()
    return f"{utc:%Y-%m-%d %H:%M} UTC ({local:%Y-%m-%d %H:%M} local)"


def main() -> int:
    parser = argparse.ArgumentParser(description="Phase 5: storage report (read-only)")
    parser.add_argument("--config", type=Path, default=DEFAULT_CONFIG_PATH)
    parser.add_argument("--max-gb", type=float, help="simulate a different storage.max_storage_gb")
    parser.add_argument("--min-free-gb", type=float, help="simulate a different storage.min_free_gb")
    args = parser.parse_args()
    try:
        settings, _ = load_settings(args.config, create_if_missing=False)
    except ConfigError as exc:
        print(f"Configuration error: {exc}", file=sys.stderr)
        return 2

    s = settings.storage
    root = settings.recordings_dir
    max_gb = args.max_gb if args.max_gb is not None else s.max_storage_gb
    min_free_gb = args.min_free_gb if args.min_free_gb is not None else s.min_free_gb
    if not root.is_dir():
        print(f"Recordings folder {root} does not exist yet.")
        return 1

    files = scan_segments(root)
    complete = [f for f in files if not f.partial]
    partials = [f for f in files if f.partial]
    recovered = [f for f in complete if f.path.name.endswith(RECOVERED_SUFFIX)]
    used = sum(f.size for f in files)
    disk = shutil.disk_usage(root)
    mount = mount_point(root)
    seg_len = settings.recording.segment_seconds

    print(f"Recordings folder : {root}")
    print(f"Stored on         : {mount_device(mount)}, mounted at {mount}")
    print(f"Disk              : {disk.total / GB:.1f} GB total, {disk.free / GB:.1f} GB free")
    print(f"Limits            : max {max_gb:g} GB, keep {min_free_gb:g} GB free, max age "
          f"{f'{s.max_age_days} days' if s.max_age_days else 'off'}, required mount "
          f"{s.required_mount or 'none'}" + ("   (simulated)" if args.max_gb or args.min_free_gb else ""))
    print(f"Recordings        : {used / GB:.2f} GB in {len(complete)} segments "
          f"({len(recovered)} recovered, {len(partials)} unfinished)")
    print(f"Oldest / newest   : {fmt_time(complete[0].start if complete else None)} / "
          f"{fmt_time(complete[-1].start if complete else None)}")

    full = [f for f in complete if f.start % seg_len == 0 and not f.path.name.endswith(RECOVERED_SUFFIX)]
    sample = full[-BITRATE_SAMPLE_SEGMENTS - 1:-1] or full[-BITRATE_SAMPLE_SEGMENTS:]
    if sample:
        bytes_per_s = sum(f.size for f in sample) / (len(sample) * seg_len)
        capacity = min(max_gb * GB, used + disk.free - min_free_gb * GB)
        hours = capacity / bytes_per_s / 3600 if bytes_per_s else 0
        print(f"Average bitrate   : {bytes_per_s * 8 / 1e6:.2f} Mbit/s (last {len(sample)} full segments)")
        print(f"Estimated history : {capacity / GB:.2f} GB usable = about {hours:.1f} h ({hours / 24:.1f} days)")
    else:
        print("Average bitrate   : not enough full-length segments yet")

    plan = plan_deletions(files, now=time.time(), max_bytes=int(max_gb * GB),
                          min_free_bytes=int(min_free_gb * GB), free_bytes=disk.free,
                          max_age_days=s.max_age_days, protected=set())
    print(f"Would delete now  : {len(plan)} segment(s)")
    for f, reason in plan[:10]:
        print(f"    {f.rel_path} ({f.size / 1e6:.1f} MB): {reason}")
    if len(plan) > 10:
        print(f"    ... and {len(plan) - 10} more")
    return 0


if __name__ == "__main__":
    sys.exit(main())
EOF
```

```bash
cat > ~/surveillance/tools/set_setting.py <<'EOF'
#!/usr/bin/env python3
"""Change one setting safely (validated, atomic write).

    python3 ~/surveillance/tools/set_setting.py storage.max_storage_gb 45
    python3 ~/surveillance/tools/set_setting.py storage.required_mount /mnt/cctv
    python3 ~/surveillance/tools/set_setting.py --show

Values are parsed as JSON when possible (numbers, true/false), otherwise used as text.
Restart the recorder afterwards for the change to take effect.
"""
from __future__ import annotations

import argparse
import json
import sys
from dataclasses import asdict
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent.parent))

from app.config import DEFAULT_CONFIG_PATH, ConfigError, load_settings, save_settings, settings_from_dict  # noqa: E402


def main() -> int:
    parser = argparse.ArgumentParser(description="Change one setting")
    parser.add_argument("key", nargs="?", help="section.name, e.g. storage.max_storage_gb")
    parser.add_argument("value", nargs="?")
    parser.add_argument("--config", type=Path, default=DEFAULT_CONFIG_PATH)
    parser.add_argument("--show", action="store_true", help="print all current settings")
    args = parser.parse_args()

    try:
        settings, _ = load_settings(args.config)
    except ConfigError as exc:
        print(f"Configuration error: {exc}", file=sys.stderr)
        return 2
    data = asdict(settings)
    if args.show or args.key is None:
        print(json.dumps(data, indent=2))
        return 0
    if args.value is None or args.key.count(".") != 1:
        parser.error("usage: set_setting.py section.name value")

    section, name = args.key.split(".")
    if section not in data or name not in data[section]:
        print(f"Unknown setting: {args.key}", file=sys.stderr)
        return 2
    try:
        value = json.loads(args.value)
    except json.JSONDecodeError:
        value = args.value
    old = data[section][name]
    data[section][name] = value
    try:
        save_settings(settings_from_dict(data), args.config)
    except ConfigError as exc:
        print(f"Not saved: {exc}", file=sys.stderr)
        return 2
    print(f"{args.key}: {old!r} -> {value!r}  (restart the recorder to apply)")
    return 0


if __name__ == "__main__":
    sys.exit(main())
EOF
```

## Test procedure

Run everything as `ysak` from `~/surveillance`. You'll need two SSH windows for test A.

**0. Add the new storage section to your settings file and look at the report.**

```bash
cd ~/surveillance
python3 tools/set_setting.py storage.max_storage_gb 45
python3 tools/set_setting.py recording.segment_seconds 60
python3 tools/phase5_storage_report.py
```

Note the **"GB free"** number on the `Disk` line; test C uses it.

**A. Crash recovery (default limits).** In window 1, run `python3 -m app.main`. After about 90 seconds, kill it from window 2:

```bash
pkill -9 -f "^python3 -m app.main"
```

Start `python3 -m app.main` again in window 1. **Within about 30–90 seconds** you should see a `Recovered unfinished segment …` line. Recovery waits until the file has been untouched for 60 seconds, so it can never grab a file that is still being written. Stop with Ctrl+C.

**B. Size limit.** Set the limit to 0.03 GB, which is about 3 one-minute segments at your bitrate. Run for about 5 minutes:

```bash
python3 tools/set_setting.py storage.max_storage_gb 0.03
python3 -m app.main
```

You should see `Deleted … over the 0.03 GB limit` lines. Stop with Ctrl+C, then check:

```bash
python3 tools/phase5_storage_report.py      # Recordings should be about 0.03 GB or less
python3 tools/phase4_check_segments.py      # the remaining segments still pass
```

**C. Free-space safety and the emergency pause.** This deletes your test recordings. Let **F** be the "GB free" number from step 0 (say 40).

- **Low:** run `python3 tools/set_setting.py storage.min_free_gb 45`, using F + 5. Then run `python3 -m app.main` for about 2 minutes. Expect `Storage low: …` warnings and deletions, **while recording continues**.
- **Critical:** run `python3 tools/set_setting.py storage.min_free_gb 100`, using any value above 2 × F. Run it again for about 1 minute. Expect `Storage critical: … PAUSED` and `Recording paused: storage is critically full`, and **no new segments**. Stop with Ctrl+C.

**D. Unmounted SSD guard.** Set up a pretend SSD location, try to start, check that nothing was created on the SD card, then undo it in this order:

```bash
python3 tools/set_setting.py paths.recordings_dir /mnt/cctv/recordings
python3 tools/set_setting.py storage.required_mount /mnt/cctv
python3 -m app.main           # should refuse and retry every 5, 10, 20 s; press Ctrl+C
ls /mnt/cctv                  # should say: No such file or directory
python3 tools/set_setting.py storage.required_mount '""'
python3 tools/set_setting.py paths.recordings_dir recordings
```

**E. Restore the normal settings:**

```bash
python3 tools/set_setting.py storage.max_storage_gb 45
python3 tools/set_setting.py storage.min_free_gb 2
python3 tools/set_setting.py recording.segment_seconds 300
python3 tools/set_setting.py --show
```

## Expected output

For test A, after the restart:

```text
INFO Main: Surveillance recorder 0.5.0 starting (...)
INFO StorageManager: Storage usage 0.03 GB / 45 GB, 40.1 GB free, 3 segments, oldest 2026-09-23 15:20 UTC
INFO Recorder: Started recording to /home/ysak/surveillance/recordings (60 s segments, ...)
WARNING StorageManager: Recovered unfinished segment 2026-09-23/15-24-00Z.mp4.partial -> 15-24-00Z.recovered.mp4: 31.9 s playable, removed 350 KiB of incomplete data
```

For test B:

```text
INFO Recorder: Segment completed: 2026-09-23/15-31-00Z.mp4 ... size=9.4MB
INFO StorageManager: Deleted 2026-09-23/15-27-00Z.mp4 (9.4 MB): over the 0.03 GB limit
```

The Phase 4 checker reports a gap before each `.recovered.mp4`, and the lost time is only the last half-written fragment, about 2 s at most. You can also copy a `.recovered.mp4` to your laptop and play it.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `Not saved: …` from `set_setting.py` | The message says what's wrong. For test D, the two settings must be changed in the order shown. |
| No `Recovered …` line after test A | Wait 90 s. If there's still nothing, check that a `.partial` exists: `ls recordings/*/`. |
| `ffprobe is not available` | `sudo apt install -y ffmpeg` |
| `Maximum storage … does not fit on this disk` | Informational: your SD card can't hold 45 GB while keeping 2 GB free, so the free-space rule applies first. It goes away with a bigger disk or a smaller limit. |

**Please send me the `Recovered …` line from test A, a few `Deleted …` lines and the report from test B, and the log lines from tests C and D.**

After test E, the recorder can safely run for long periods, because the SD card can no longer fill up. It still only runs while your SSH session is open; it becomes an automatically started service in Phase 13. Phase 6 adds motion detection on the 640×480 low-resolution stream.
