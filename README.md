Phase 4 (continuous recording in segments) is ready for you to run on the Pi. I tested the real code here against a fake camera that feeds genuine H.264 frames, but it hasn't touched your real camera yet.

I'm treating the still image you saw in VLC as a VLC display quirk, so recordings use fragmented MP4 (the format that survives a power cut). If you run the fingerprint commands from my last message at some point, I'd still like to see the output.

## What I tested here

- **Segments line up with the clock.** With 60-second test segments, files were named `13-45-00Z.mp4`, `13-46-00Z.mp4` and so on. Each was exactly 60.0 s and 900 frames, started on a keyframe, and had no timestamp gaps. Back-to-back segments joined with 0.0 s missing.
- **A frozen camera is detected.** When I stopped the fake camera's frames, the recorder noticed after 11 s, saved the segment in progress, and restarted the camera pipeline.
- **A busy camera is retried.** While the camera was "busy", reopening was retried after 5, 10 and 20 s, and recording resumed by itself once the camera was free.
- **A hard crash loses almost nothing.** After `kill -9`, the unfinished segment stayed as a `.partial` file with 21.5 s still playable. On restart the recorder warned about it and kept it for recovery.
- **Ctrl+C stops cleanly.** The current segment was saved as a normal `.mp4`.
- **Bad settings are rejected with a clear message.** I tried a width that isn't a multiple of 16, a misspelled setting, a 45-second segment length, the text `"yes"` instead of `true`, a camera name containing `<script>`, and a size over the encoder's limit.

## How it works

- **Its own segmenting output instead of Picamera2's `SplittableOutput`.** `SplittableOutput` waits for the next keyframe with no timeout, so a stalled camera would hang the recorder forever. My `SegmentingOutput` switches files inside the encoder's own thread at the first keyframe after each boundary, so it can never wait on a frame that won't come.
- **The encoder thread never waits for the disk.** Flushing a closed segment to disk and renaming it `.partial` → `.mp4` happens in a separate thread.
- **Every file starts at time zero,** so browsers show 0:00 to 5:00 for each segment.
- **Naming:** the first segment after a start is named after its real start time and runs until the next 5-minute boundary, so it is shorter. After that, names are aligned: `…/2026-09-23/13-05-00Z.mp4`. The exact start time is kept in the segment's details.
- **Supervisor:** if the camera can't be opened, or frames stop for 10 s, the pipeline is restarted, waiting 5, 10, 20… up to 60 s between attempts.
- **Clock monitor:** each segment is marked with whether the clock was NTP-synchronised when it started (the Pi 4 has no battery-backed clock).
- **Settings** are a validated JSON file, created with defaults on first run. **Logs** go to `logs/surveillance.log` (5 files × 5 MB maximum) and the console, in the format `2026-09-23 21:35:00 INFO Recorder: …`.
- **Not yet built:** there is no storage limit until Phase 5 and no database until Phase 7. Until then, the filename convention (`.partial` means still being written, `.mp4` means complete) is how the rest of the system will tell them apart.

> **Important:** recordings go to `~/surveillance/recordings` on the SD card with **no size limit** until Phase 5. At your measured 1.25 Mbit/s that's about 13.5 GB per day, so don't leave it running for days yet.

## Files

| File | Purpose |
|---|---|
| `app/__init__.py` | Version number |
| `app/fileutil.py` | Crash-safe `fsync` and atomic-write helpers |
| `app/config.py` | Settings schema, validation, load and save |
| `app/logging_setup.py` | Log format and rotation |
| `app/clock.py` | NTP sync monitor |
| `app/camera.py` | Opens and configures Picamera2 |
| `app/recorder.py` | H.264 encoder, segment switching and finalising |
| `app/main.py` | Entry point, supervisor and signal handling |
| `tools/phase4_check_segments.py` | Checks the recorded segments |
| `config/settings.json` | Created automatically on first run |

## Install

Everything needed is already installed (`python3-picamera2`, `python3-av`, `ffmpeg`). Create the folder:

```bash
mkdir -p ~/surveillance/app ~/surveillance/tools
cd ~/surveillance
```

Then paste each block below into the terminal, one at a time. Each block writes one complete file.

```bash
cat > ~/surveillance/app/__init__.py <<'EOF'
"""Raspberry Pi surveillance camera."""

__version__ = "0.4.0"
EOF
```

```bash
cat > ~/surveillance/app/fileutil.py <<'EOF'
"""Crash-safe file helpers."""
from __future__ import annotations

import os
from pathlib import Path


def fsync_directory(directory: Path) -> None:
    """Persist a directory entry (a new or renamed file) across power loss."""
    fd = os.open(directory, os.O_RDONLY | os.O_DIRECTORY)
    try:
        os.fsync(fd)
    finally:
        os.close(fd)


def fsync_file(path: Path) -> None:
    with open(path, "rb") as handle:
        os.fsync(handle.fileno())


def atomic_write_bytes(path: Path, data: bytes, mode: int = 0o640) -> None:
    """Replace `path` so that readers see either the old or the new content, never a mix."""
    path.parent.mkdir(parents=True, exist_ok=True)
    tmp = path.with_name(f".{path.name}.tmp")
    try:
        fd = os.open(tmp, os.O_WRONLY | os.O_CREAT | os.O_TRUNC, mode)
        with os.fdopen(fd, "wb") as handle:
            handle.write(data)
            handle.flush()
            os.fsync(handle.fileno())
        os.replace(tmp, path)
    except BaseException:
        tmp.unlink(missing_ok=True)
        raise
    fsync_directory(path.parent)
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
    paths: PathSettings = field(default_factory=PathSettings)
    logging: LoggingSettings = field(default_factory=LoggingSettings)

    def validate(self) -> None:
        for section in fields(self):
            getattr(self, section.name).validate()

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
cat > ~/surveillance/app/logging_setup.py <<'EOF'
"""Structured logging: timestamp, level, component, message; size-capped rotation."""
from __future__ import annotations

import json
import logging
import sys
from datetime import datetime, timezone
from logging.handlers import RotatingFileHandler
from pathlib import Path

from app.config import LoggingSettings

TEXT_FORMAT = "%(asctime)s %(levelname)s %(name)s: %(message)s"
TEXT_DATE_FORMAT = "%Y-%m-%d %H:%M:%S"
LOG_FILE_NAME = "surveillance.log"


class JsonFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        entry = {
            "ts": datetime.fromtimestamp(record.created, timezone.utc).isoformat(timespec="milliseconds"),
            "level": record.levelname,
            "component": record.name,
            "message": record.getMessage(),
        }
        if record.exc_info:
            entry["exception"] = self.formatException(record.exc_info)
        return json.dumps(entry, ensure_ascii=False)


def setup_logging(settings: LoggingSettings, log_dir: Path) -> Path | None:
    """Configure the root logger. Returns the log file path, or None if it is not writable."""
    formatter: logging.Formatter = (JsonFormatter() if settings.format == "json"
                                    else logging.Formatter(TEXT_FORMAT, TEXT_DATE_FORMAT))
    root = logging.getLogger()
    root.setLevel(settings.level)
    for handler in list(root.handlers):
        root.removeHandler(handler)
        handler.close()

    if settings.console:
        console = logging.StreamHandler(sys.stderr)
        console.setFormatter(formatter)
        root.addHandler(console)

    log_file: Path | None = log_dir / LOG_FILE_NAME
    try:
        log_dir.mkdir(parents=True, exist_ok=True)
        file_handler = RotatingFileHandler(log_file, maxBytes=settings.max_bytes,
                                           backupCount=settings.backup_count, encoding="utf-8")
    except OSError as exc:
        log_file = None
        if not settings.console:
            fallback = logging.StreamHandler(sys.stderr)
            fallback.setFormatter(formatter)
            root.addHandler(fallback)
        logging.getLogger("Logging").error("Cannot write log file in %s: %s", log_dir, exc.strerror)
    else:
        file_handler.setFormatter(formatter)
        root.addHandler(file_handler)

    logging.captureWarnings(True)
    # Picamera2 logs every configuration change at INFO; keep only its warnings and errors.
    logging.getLogger("picamera2").setLevel(max(logging.WARNING, root.level))
    return log_file
EOF
```

```bash
cat > ~/surveillance/app/clock.py <<'EOF'
"""Tracks whether the system clock is NTP-synchronised.

The Pi 4 has no battery-backed clock: after a power cut without internet it resumes from the
last saved time, so recordings made before synchronisation are flagged.
"""
from __future__ import annotations

import logging
import subprocess
import threading
from pathlib import Path

log = logging.getLogger("Clock")

TIMESYNCD_FLAG = Path("/run/systemd/timesync/synchronized")


def query_synchronized() -> bool | None:
    try:
        result = subprocess.run(["timedatectl", "show", "-p", "NTPSynchronized", "--value"],
                                capture_output=True, text=True, timeout=5, check=False)
        value = result.stdout.strip()
        if value in ("yes", "no"):
            return value == "yes"
    except (OSError, subprocess.TimeoutExpired):
        pass
    return True if TIMESYNCD_FLAG.exists() else None


class ClockMonitor:
    def __init__(self, interval_seconds: float = 60.0) -> None:
        self._interval = interval_seconds
        self._synced: bool | None = None
        self._stop = threading.Event()
        self._thread = threading.Thread(target=self._run, name="clock-monitor", daemon=True)

    @property
    def synchronized(self) -> bool | None:
        return self._synced

    def start(self) -> None:
        self._update()
        self._thread.start()

    def stop(self) -> None:
        self._stop.set()
        if self._thread.is_alive():
            self._thread.join(timeout=5)

    def _run(self) -> None:
        while not self._stop.wait(self._interval):
            self._update()

    def _update(self) -> None:
        synced = query_synchronized()
        if synced != self._synced:
            if synced:
                log.info("System clock is NTP-synchronised")
            else:
                log.warning("System clock is NOT synchronised (%s); new recordings are flagged "
                            "until it is", "unknown" if synced is None else "no NTP sync yet")
        self._synced = synced
EOF
```

```bash
cat > ~/surveillance/app/camera.py <<'EOF'
"""Picamera2 wrapper: opens and configures the camera with a main + lores stream."""
from __future__ import annotations

import logging

from app.config import CameraSettings

log = logging.getLogger("Camera")


class CameraError(RuntimeError):
    pass


class Camera:
    def __init__(self, settings: CameraSettings) -> None:
        self.settings = settings
        self.picam2 = None

    def open(self) -> None:
        try:
            from libcamera import Transform
            from picamera2 import Picamera2
        except ImportError as exc:
            raise CameraError(f"Picamera2 is not installed ({exc})") from exc

        s = self.settings
        cameras = Picamera2.global_camera_info()
        if s.camera_num >= len(cameras):
            raise CameraError(f"camera {s.camera_num} not detected ({len(cameras)} camera(s) found)")
        try:
            self.picam2 = Picamera2(s.camera_num)
        except (RuntimeError, IndexError) as exc:
            raise CameraError(f"cannot open camera {s.camera_num} (in use by another program?): {exc}") from exc

        try:
            size = (s.width, s.height)
            sensor_size = self._matching_sensor_size(size)
            config = self.picam2.create_video_configuration(
                main={"size": size, "format": "YUV420"},
                lores={"size": (s.lores_width, s.lores_height), "format": "YUV420"},
                sensor={"output_size": sensor_size} if sensor_size else {},
                controls={"FrameRate": s.framerate},
                transform=Transform(hflip=s.rotate180, vflip=s.rotate180),
            )
            self.picam2.configure(config)
        except Exception as exc:
            self.close()
            raise CameraError(f"cannot configure camera: {exc}") from exc

        model = self.picam2.camera_properties.get("Model", "unknown")
        log.info("Opened %s: main %dx%d, lores %dx%d, %g fps, sensor mode %s", model, s.width, s.height,
                 s.lores_width, s.lores_height, s.framerate,
                 f"{sensor_size[0]}x{sensor_size[1]}" if sensor_size else "auto")

    def _matching_sensor_size(self, size: tuple[int, int]) -> tuple[int, int] | None:
        """Use the sensor mode with exactly the recording size (e.g. OV5647 1296x972 full view)."""
        for mode in self.picam2.sensor_modes:
            if tuple(mode["size"]) == size:
                return size
        return None

    def close(self) -> None:
        if self.picam2 is None:
            return
        try:
            self.picam2.close()
        except Exception as exc:  # noqa: BLE001 - closing must never raise
            log.debug("Error while closing camera: %s", exc)
        self.picam2 = None
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
                 on_closed: Callable[[_OpenSegment], None]) -> None:
        super().__init__()
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

    def __init__(self, settings: Settings, clock: ClockMonitor) -> None:
        self.settings = settings
        self.recordings_dir = settings.recordings_dir
        self._clock = clock
        self._camera: Camera | None = None
        self._output: SegmentingOutput | None = None
        self._recording = False
        self._started_monotonic = 0.0
        self._finalize_queue: queue.Queue[_OpenSegment | None] = queue.Queue()
        self._finalizer = threading.Thread(target=self._finalize_loop, name="segment-finalizer", daemon=True)
        self._listeners: list[Callable[[Segment], None]] = []
        self.last_segment: Segment | None = None

    def add_listener(self, callback: Callable[[Segment], None]) -> None:
        self._listeners.append(callback)

    @property
    def current_segment(self) -> str | None:
        return self._output.current_rel_path if self._output else None

    def start(self) -> None:
        cam = self.settings.camera
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
                                        self._finalize_queue.put)
        self._camera.picam2.start_recording(encoder, self._output)
        self._recording = True
        self._started_monotonic = time.monotonic()
        log.info("Started recording to %s (%d s segments, %.2f Mbit/s, keyframe every %g s)",
                 self.recordings_dir, self.settings.recording.segment_seconds, cam.bitrate / 1e6,
                 cam.keyframe_seconds)

    def stop(self) -> None:
        was_recording = self._recording
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

import argparse
import logging
import os
import signal
import sys
import threading
import time
from pathlib import Path

from app import __version__
from app.camera import CameraError
from app.clock import ClockMonitor
from app.config import DEFAULT_CONFIG_PATH, ConfigError, Settings, load_settings
from app.logging_setup import setup_logging
from app.recorder import Recorder

log = logging.getLogger("Main")

BACKOFF_INITIAL_S = 5
BACKOFF_MAX_S = 60
HEALTHY_RESET_S = 120
PARTIAL_GLOB = "*/*.mp4.partial"


def parse_args(argv: list[str] | None = None) -> argparse.Namespace:
    parser = argparse.ArgumentParser(description="Raspberry Pi surveillance recorder")
    parser.add_argument("--config", type=Path,
                        default=Path(os.environ.get("SURVEILLANCE_CONFIG", DEFAULT_CONFIG_PATH)),
                        help=f"settings file (default {DEFAULT_CONFIG_PATH})")
    return parser.parse_args(argv)


def report_leftover_partials(settings: Settings) -> None:
    leftovers = sorted(settings.recordings_dir.glob(PARTIAL_GLOB))
    if leftovers:
        log.warning("%d unfinished segment(s) from a previous run (crash or power cut), newest: %s. "
                    "They are kept for recovery.", len(leftovers), leftovers[-1].name)


def run(settings: Settings, stop: threading.Event) -> None:
    clock = ClockMonitor()
    clock.start()
    report_leftover_partials(settings)
    backoff = BACKOFF_INITIAL_S
    try:
        while not stop.is_set():
            recorder = Recorder(settings, clock)
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
cat > ~/surveillance/tools/phase4_check_segments.py <<'EOF'
#!/usr/bin/env python3
"""Phase 4: verify the segmented recordings written by the recorder.

Run from anywhere (while the recorder runs, or after stopping it):

    python3 ~/surveillance/tools/phase4_check_segments.py [--date YYYY-MM-DD] [--last N]

For every completed segment it checks: starts on a keyframe, no internal timestamp gaps,
frame rate, duration close to the configured segment length. It also checks that consecutive
segments leave no missing time, and probes any leftover .partial files.
"""
from __future__ import annotations

import argparse
import json
import subprocess
import sys
from datetime import datetime, timedelta, timezone
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent.parent))

from app.config import DEFAULT_CONFIG_PATH, ConfigError, load_settings  # noqa: E402

CONTINUITY_TOLERANCE_S = 3.0


def ffprobe_packets(path: Path) -> list[tuple[float, bool]]:
    result = subprocess.run(
        ["ffprobe", "-v", "error", "-select_streams", "v:0",
         "-show_entries", "packet=pts_time,flags", "-of", "csv=p=0", str(path)],
        capture_output=True, text=True, timeout=300, check=False)
    packets = []
    for line in result.stdout.splitlines():
        pts, _, flags = line.partition(",")
        try:
            packets.append((float(pts), "K" in flags))
        except ValueError:
            continue
    return sorted(packets)


def ffprobe_size(path: Path) -> tuple[int | None, int | None]:
    result = subprocess.run(
        ["ffprobe", "-v", "error", "-select_streams", "v:0",
         "-show_entries", "stream=width,height", "-of", "json", str(path)],
        capture_output=True, text=True, timeout=60, check=False)
    try:
        stream = json.loads(result.stdout or "{}").get("streams", [{}])[0]
    except (json.JSONDecodeError, IndexError):
        return None, None
    return stream.get("width"), stream.get("height")


def nominal_start(path: Path) -> datetime | None:
    """Recover the UTC start time encoded in <YYYY-MM-DD>/<HH-MM-SS>Z[-n].mp4."""
    stem = path.name.split(".")[0].split("Z")[0]
    try:
        return datetime.strptime(f"{path.parent.name} {stem}", "%Y-%m-%d %H-%M-%S").replace(tzinfo=timezone.utc)
    except ValueError:
        return None


def main() -> int:
    parser = argparse.ArgumentParser(description="Phase 4: check segmented recordings")
    parser.add_argument("--config", type=Path, default=DEFAULT_CONFIG_PATH)
    parser.add_argument("--date", help="UTC day folder to check (default: newest)")
    parser.add_argument("--last", type=int, default=0, help="only check the newest N segments")
    args = parser.parse_args()

    try:
        settings, _ = load_settings(args.config, create_if_missing=False)
    except ConfigError as exc:
        print(f"Configuration error: {exc}", file=sys.stderr)
        return 2
    root = settings.recordings_dir
    seg_len = settings.recording.segment_seconds
    fps = settings.camera.framerate
    size = (settings.camera.width, settings.camera.height)

    days = sorted(p for p in root.iterdir() if p.is_dir()) if root.is_dir() else []
    if not days:
        print(f"No recordings found in {root}")
        return 1
    day = root / args.date if args.date else days[-1]
    segments = sorted(day.glob("*.mp4"), key=lambda p: (nominal_start(p) or datetime.min, p.name))
    partials = sorted(day.glob("*.mp4.partial"))
    if args.last:
        segments = segments[-args.last:]
    print(f"Checking {len(segments)} segment(s) in {day} (segment length {seg_len} s, {fps:g} fps)\n")

    failures = warnings = 0
    checked = []
    for path in segments:
        packets = ffprobe_packets(path)
        width, height = ffprobe_size(path)
        if len(packets) < 2:
            print(f"[FAIL] {path.name}: unreadable ({len(packets)} frames)")
            failures += 1
            continue
        times = [t for t, _ in packets]
        gaps = sum(1 for a, b in zip(times, times[1:]) if b - a > 1.5 / fps)
        duration = times[-1] - times[0] + 1 / fps
        measured_fps = (len(times) - 1) / (times[-1] - times[0])
        problems = []
        if not packets[0][1]:
            problems.append("does not start with a keyframe")
        if gaps:
            problems.append(f"{gaps} timestamp gap(s)")
        if (width, height) != size:
            problems.append(f"resolution {width}x{height}")
        if abs(measured_fps - fps) > 0.05 * fps:
            problems.append(f"{measured_fps:.2f} fps")
        status = "FAIL" if problems else "PASS"
        failures += bool(problems)
        mb = path.stat().st_size / 1e6
        print(f"[{status}] {path.name}: {duration:6.1f} s, {len(packets)} frames, {measured_fps:.2f} fps, "
              f"{mb:.1f} MB, {mb * 8 / duration:.2f} Mbit/s" + (f"  <- {', '.join(problems)}" if problems else ""))
        checked.append((path, nominal_start(path), duration))

    print("\nContinuity (a new segment should start where the previous one ended):")
    runs = 0
    for (prev, prev_start, prev_dur), (cur, cur_start, _) in zip(checked, checked[1:]):
        if prev_start is None or cur_start is None:
            continue
        missing = (cur_start - prev_start).total_seconds() - prev_dur
        # The first segment after a (re)start is named after its real start time, not a boundary.
        restarted = cur_start.timestamp() % seg_len != 0
        if restarted:
            print(f"  [INFO] {prev.name} -> {cur.name}: recorder stopped/restarted, "
                  f"{timedelta(seconds=max(0, round(missing)))} not in completed segments")
            runs += 1
        elif abs(missing) <= CONTINUITY_TOLERANCE_S:
            print(f"  [PASS] {prev.name} -> {cur.name}: {missing:+.1f} s")
        else:
            print(f"  [WARN] {prev.name} -> {cur.name}: {missing:+.1f} s unaccounted for")
            warnings += 1

    if partials:
        print("\nUnfinished .partial files (the newest one is normal while the recorder is running):")
        for path in partials:
            packets = ffprobe_packets(path)
            playable = packets[-1][0] - packets[0][0] if len(packets) > 1 else 0.0
            print(f"  {path.name}: {path.stat().st_size / 1e6:.1f} MB, {playable:.1f} s playable")

    print(f"\nFAIL={failures} WARN={warnings} stopped-and-restarted={runs}")
    if failures:
        print("RESULT: PROBLEMS FOUND")
        return 1
    print("RESULT: SEGMENTS OK")
    return 0


if __name__ == "__main__":
    sys.exit(main())
EOF
```

## Test procedure

Run everything **as `ysak`**, from `~/surveillance`. You'll need **two SSH windows** for the crash and camera-busy tests.

**0. First run creates the settings file.** Stop it with Ctrl+C once you see `Started recording`:

```bash
cd ~/surveillance
python3 -m app.main
```

Then switch to **60-second segments for testing**:

```bash
sed -i 's/"segment_seconds": 300/"segment_seconds": 60/' config/settings.json
grep segment_seconds config/settings.json
```

**A. Normal recording, about 5 minutes.**

```bash
python3 -m app.main
```

Wait for at least four `Segment completed` lines, then press Ctrl+C and check the files:

```bash
python3 tools/phase4_check_segments.py
```

**B. Crash test.** In window 1, start `python3 -m app.main`. After about 90 seconds, run this in window 2:

```bash
pkill -9 -f "^python3 -m app.main"
```

Back in window 1, start the recorder again. It should warn `1 unfinished segment(s) from a previous run`. Let it run for about 2 minutes, press Ctrl+C, then run the checker again.

**C. Camera-busy recovery.** In window 2, occupy the camera:

```bash
rpicam-hello -n -t 0
```

In window 1, run `python3 -m app.main`. It should report `Could not start recording: cannot open camera 0 … Retrying in 5 s`, then 10 s, then 20 s. Press Ctrl+C in window 2 to free the camera; recording should start within one retry. Then press Ctrl+C in window 1.

**D. Settings validation.** Put in a bad value, try to start, then restore it:

```bash
sed -i 's/"framerate": 15.0/"framerate": 99/' config/settings.json
python3 -m app.main          # should refuse to start
sed -i 's/"framerate": 99/"framerate": 15.0/' config/settings.json
```

**E. Go back to 5-minute segments and check the log:**

```bash
sed -i 's/"segment_seconds": 60/"segment_seconds": 300/' config/settings.json
tail -n 20 logs/surveillance.log
```

**F. Optional playback check.** Copy one segment to your laptop (run this on the laptop) and play it:

```bash
scp 'ysak@ysak.local:~/surveillance/recordings/*/*.mp4' .
```

**Optional cleanup when you're done:** `rm -rf ~/surveillance/recordings/*`

## Expected output

For test A, the log shows (in your local time, UTC+8):

```text
2026-09-23 21:50:10 INFO Main: Surveillance recorder 0.4.0 starting (config /home/ysak/surveillance/config/settings.json, log /home/ysak/surveillance/logs/surveillance.log)
2026-09-23 21:50:10 INFO Clock: System clock is NTP-synchronised
2026-09-23 21:50:12 INFO Camera: Opened ov5647: main 1296x972, lores 640x480, 15 fps, sensor mode 1296x972
2026-09-23 21:50:12 INFO Recorder: Started recording to /home/ysak/surveillance/recordings (60 s segments, 2.50 Mbit/s, keyframe every 2 s)
2026-09-23 21:51:01 INFO Recorder: Segment completed: 2026-09-23/13-50-12Z.mp4 start=2026-09-23T13:50:12.874+00:00 duration=48.0s frames=720 size=7.5MB
2026-09-23 21:52:01 INFO Recorder: Segment completed: 2026-09-23/13-51-00Z.mp4 start=2026-09-23T13:51:01.027+00:00 duration=60.0s frames=900 size=9.4MB
```

The first segment is shorter because it only runs until the next minute boundary. Aligned segments start 0–2 s after the boundary, because switching waits for a keyframe.

The checker should show:

```text
[PASS] 13-51-00Z.mp4:   60.0 s, 900 frames, 15.00 fps, 9.4 MB, 1.25 Mbit/s
...
Continuity (a new segment should start where the previous one ended):
  [PASS] 13-50-12Z.mp4 -> 13-51-00Z.mp4: +0.0 s
...
RESULT: SEGMENTS OK
```

After test B, the stop is shown as `[INFO] … recorder stopped/restarted`, and the leftover `.partial` is listed with its playable seconds. Phase 5 will recover these files automatically.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `ModuleNotFoundError: No module named 'app'` | Run it from `~/surveillance`, exactly as `python3 -m app.main`. |
| `Configuration error: …` | The message names the setting at fault. Fix it in `config/settings.json`, or delete the file to recreate the defaults. |
| `Recording unhealthy: no frames …` keeps repeating | Send me the log and the output of `vcgencmd get_throttled`. |
| A checker `[WARN] … unaccounted for` between aligned segments | Send me the checker output. Frames were lost at a switch between files, which shouldn't happen. |
| `Permission denied` on `recordings` or `logs` | Earlier root use probably left root-owned files. Run `sudo chown -R ysak:ysak ~/surveillance`. |

**Please send me the checker output from tests A and B, and the log lines from test C.** Phase 5 then adds the storage limit (45 GB), the emergency free-space threshold, deleting the oldest finished segment first (never the one being written), and automatic recovery of `.partial` files at startup.
