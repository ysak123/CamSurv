Your Phase 4 results showed one real problem, and I've fixed it. The first segment after every start had 1–2 short gaps of dropped frames. Everything else passed: every other segment had exactly 900 frames at 15.00 fps with 0.0 s missing between files. The crash recovery and camera-busy retries also worked exactly as intended.

## What caused the gaps

Picamera2 only loads its video-file library (PyAV) the first time it creates an MP4 file. In my recorder, that happens inside the encoder thread on the very first frame, which stalls it. The camera drops frames when the encoder stalls. In Phase 3 the library was loaded before recording started, which is why Phase 3 had no gaps. Loading it takes about 46 ms on this fast machine; on a Pi 4 reading from an SD card it's likely several hundred milliseconds, which is enough to drop 4–7 frames.

I reproduced it here with a simulated camera that drops frames the way real hardware does:

| Version | First segment | Other segments | Aligned segments start at |
|---|---|---|---|
| Old | FAIL: 1 gap at 0.0 s (+400 ms) | PASS | about 1–2 s after the boundary |
| New | PASS, no gaps | PASS (900 frames each) | `14:34:00.048`, about 50 ms after |

## What changed in version 0.4.1

1. **The library is loaded before the camera starts.** This is the fix for the gaps.
2. **A keyframe is requested exactly at each boundary.** Before, every aligned segment waited for the next regular keyframe; yours consistently started about 1.9 s late (`14:06:01.957`, `14:07:01.942`…). Now a `…-05-00Z.mp4` file really starts at about :00.1. That will matter when motion events are matched to recordings.
3. **libcamera's verbose INFO lines are hidden.** Only its warnings and errors appear now.
4. **The checker shows where each gap is,** for example `1 timestamp gap(s) at 0.0 s (+400 ms)`.
5. **Minor:** a doubled full stop is removed from the camera-busy error message, and stopping is safer if startup fails halfway.

## Update these five files

Paste each block into the terminal from `~/surveillance`. Each one replaces the whole file. `config.py`, `fileutil.py`, `clock.py` and `logging_setup.py` are unchanged.

```bash
cat > ~/surveillance/app/__init__.py <<'EOF'
"""Raspberry Pi surveillance camera."""

__version__ = "0.4.1"
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
            reason = str(exc).rstrip(".")
            raise CameraError(f"cannot open camera {s.camera_num} (in use by another program?): {reason}") from exc

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
        self._boundary_stop = threading.Event()
        self._boundary_thread: threading.Thread | None = None
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
        gaps = [(a - times[0], b - a) for a, b in zip(times, times[1:]) if b - a > 1.5 / fps]
        duration = times[-1] - times[0] + 1 / fps
        measured_fps = (len(times) - 1) / (times[-1] - times[0])
        problems = []
        if not packets[0][1]:
            problems.append("does not start with a keyframe")
        if gaps:
            where = ", ".join(f"{at:.1f} s (+{step * 1000:.0f} ms)" for at, step in gaps[:3])
            problems.append(f"{len(gaps)} timestamp gap(s) at {where}")
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

## Retest

**1. Clear the old test recordings** so the checker only sees segments from the new version. Your config should still have 60-second segments:

```bash
cd ~/surveillance
rm -rf recordings/*
grep segment_seconds config/settings.json      # should show 60
```

**2. Record for about 4 minutes, then stop with Ctrl+C:**

```bash
python3 -m app.main
```

**3. Start it again, record for about 1 minute, and stop.** This adds a second "first segment after a start", which is exactly the case that failed before.

**4. Run the checker:**

```bash
python3 tools/phase4_check_segments.py
```

**5. If it passes, go back to 5-minute segments** (this was test E):

```bash
sed -i 's/"segment_seconds": 60/"segment_seconds": 300/' config/settings.json
```

## Expected output

The console should no longer show libcamera's `[0:43:48…] INFO` lines. Aligned segments should start within about 0.1 s of the minute:

```text
2026-09-23 22:40:05 INFO Main: Surveillance recorder 0.4.1 starting (...)
2026-09-23 22:40:05 INFO Clock: System clock is NTP-synchronised
2026-09-23 22:40:06 INFO Camera: Opened ov5647: main 1296x972, lores 640x480, 15 fps, sensor mode 1296x972
2026-09-23 22:40:06 INFO Recorder: Started recording to /home/ysak/surveillance/recordings (60 s segments, ...)
2026-09-23 22:41:00 INFO Recorder: Segment completed: 2026-09-23/14-40-06Z.mp4 start=...T14:40:06.4... duration=53.6s frames=804 ...
2026-09-23 22:42:00 INFO Recorder: Segment completed: 2026-09-23/14-41-00Z.mp4 start=...T14:41:00.0... duration=60.0s frames=900 ...
```

The checker should end with `FAIL=0 WARN=0 stopped-and-restarted=1` and `RESULT: SEGMENTS OK`, with every line `[PASS]` including the first segment of each run.

**If a first segment still shows a gap,** the checker now says where it is. A gap at `0.0 s` would mean the camera itself delivers its first frames unevenly at start-up, rather than my code stalling. The fix would then be to skip the first second of frames. Please send me the checker output either way.

Once this passes, Phase 5 adds the storage limit (45 GB), the emergency free-space threshold, deleting the oldest finished segment first (never the one being written), and automatic recovery of `.partial` files after a crash.
