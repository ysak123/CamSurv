Phase 6 (motion detection) is ready to run on your Pi. Phase 5 passed, apart from one "critical storage" retest you need to do first because of a number I gave you. I tested Phase 6 here with synthetic scenes and the fake camera, not on your hardware.

## Phase 5 results

- **Test A (crash recovery):** passed. The unfinished file became `15-27-00Z.recovered.mp4` with 23.9 s playable, after cutting 204 KiB of half-written data.
- **Test B (size limit):** passed. Deletions went oldest first, and the file being written was never touched.
  - The checker's `+142.4 s unaccounted for` WARN is a checker mistake, not lost footage. You restarted at 15:30:59, the first frame arrived just after 15:31:00, so the new segment looked like a normal aligned one and the checker missed the restart. It now treats any gap over 10 s as a restart.
  - The report's 1.09 Mbit/s estimate counted a segment stopped after 37.6 s as a full minute. It now uses only complete one-minute segments.
- **Test C (free space):** the "low" part passed. The "critical" part **wasn't actually tested**, because of my instructions. The pause happens below half of `min_free_gb`; half of 100 is 50.0 GB and you had 50.23 GB free. My rule of "above 2 × free" meant above 100.46, and 100 was just under that. The retest with 120 is step 0 below.
- **Test D (unmounted SSD):** passed. It refused to record, retried, and created nothing on the SD card.
- **Test E** (restoring the normal settings) is still to do; it's part of step 0 below.

## What Phase 6 adds

**How detection works:**
1. The detector reads the 640×480 greyscale channel from the camera's low-resolution stream and halves it to 320×240.
2. It does this about 5 times per second, while the full-quality recording carries on untouched.
3. Each frame is blurred to remove sensor noise and compared with a slowly updating picture of the empty scene, called the background.
4. It then finds the **largest connected patch of changed pixels**.

Using the largest single patch rather than the total number of changed pixels means scattered noise can't add up to a false trigger.

**Settings** (in the new `motion` section):

| Setting | Default | Meaning |
|---|---|---|
| `sensitivity` | 70 | 1–100. Higher means a smaller brightness change counts. 70 means a change of 28 out of 255. |
| `min_area_percent` | 0.5 | The patch must cover this much of the frame, about 20×20 pixels. |
| `trigger_frames` | 3 | Movement must be seen in 3 analysed frames in a row (about 0.6 s). Single-frame flickers are ignored. |
| `cooldown_seconds` | 10 | An event ends only after 10 s with no movement, so one continuous movement is one event. |
| `analysis_fps` | 5 | Frames analysed per second. |
| `enabled` | true | Turns motion detection on or off. |

**Other behaviour:**
- **Lighting changes:** if more than 60 % of the frame changes at once (lights switched on, exposure jumps), the background is relearned instead of reporting motion.
- **Selective learning:** pixels that are currently changing are learned at half speed. A walking person doesn't leave a "ghost trail" that stretches the event. Someone who stops and stands still becomes part of the background after about 15 s.
- **Logs:** `Motion started (area 3.3% of the frame)` and `Motion ended after 14.2 s (peak area 7.1%)`. Each segment's log line now ends in `motion=yes` or `motion=no`. Phase 7 will store both in SQLite.
- **Robustness:** OpenCV is loaded **before** the camera starts (the same lesson as PyAV). If OpenCV is missing, recording still works, with motion detection disabled and an error in the log.

## What I tested here

**Synthetic 60-second scene with realistic sensor noise.** Results before and after the selective-learning change:

| Scenario | Before | After (shipped) |
|---|---|---|
| A single-frame blip at 10 s | ignored | ignored |
| Walking at 20–26 s, pausing 4 s, walking back 30–33 s | one event, 20.0–33.0 s | one event, **20.0–32.8 s** |
| Lights switching on | recognised, not reported as motion | same |
| Walking in, then standing still | not measured | event ends about 15 s after they stop |

**Fake camera, end to end:** a moving object from 20 to 32 s after start gave one event. The two segments overlapping it were marked `motion=yes` and the next one `motion=no`.

**Tuning tool:** it caught 11.8 s of the 12 s movement, the quiet-scene noise level was 0.00 %, and analysis took 0.3 ms per frame on this machine (expect a few ms on the Pi). Its debug image showed only the moving object in the mask, with no noise.

## Install

```bash
sudo apt install -y python3-opencv
python3 -c "import cv2; print(cv2.__version__)"
```

## Update the files

**1. Apply the update script.** It changes `__init__.py`, `config.py`, `recorder.py`, `storage_manager.py` and the two report tools. It checks that every piece of code it replaces is exactly what I gave you in Phase 5, makes `.bak` backups, and changes nothing if any check fails. I tested it on copies of your 0.5.0 files: the result is identical to what I tested, and running it twice safely aborts.

```bash
cd ~/surveillance
cat > update_to_0_6_0.py <<'EOF'
#!/usr/bin/env python3
"""Update the surveillance project from 0.5.0 to 0.6.0 (run from ~/surveillance)."""
import shutil, sys
from pathlib import Path

EDITS = [
    ('app/__init__.py',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.5.0"\n',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.6.0"\n'),
    ('app/config.py',
     '\n@dataclass(frozen=True)\nclass PathSettings:\n    recordings_dir: str = "recordings"\n',
     '\n@dataclass(frozen=True)\nclass MotionSettings:\n    enabled: bool = True\n    sensitivity: int = 70\n    min_area_percent: float = 0.5\n    cooldown_seconds: float = 10.0\n    trigger_frames: int = 3\n    analysis_fps: float = 5.0\n\n    def validate(self) -> None:\n        _check_range("motion.sensitivity", self.sensitivity, 1, 100)\n        _check_range("motion.min_area_percent", self.min_area_percent, 0.05, 50.0)\n        _check_range("motion.cooldown_seconds", self.cooldown_seconds, 1.0, 300.0)\n        _check_range("motion.trigger_frames", self.trigger_frames, 1, 20)\n        _check_range("motion.analysis_fps", self.analysis_fps, 1.0, 15.0)\n\n\n@dataclass(frozen=True)\nclass PathSettings:\n    recordings_dir: str = "recordings"\n'),
    ('app/config.py',
     '    recording: RecordingSettings = field(default_factory=RecordingSettings)\n    storage: StorageSettings = field(default_factory=StorageSettings)\n    paths: PathSettings = field(default_factory=PathSettings)\n    logging: LoggingSettings = field(default_factory=LoggingSettings)\n',
     '    recording: RecordingSettings = field(default_factory=RecordingSettings)\n    storage: StorageSettings = field(default_factory=StorageSettings)\n    motion: MotionSettings = field(default_factory=MotionSettings)\n    paths: PathSettings = field(default_factory=PathSettings)\n    logging: LoggingSettings = field(default_factory=LoggingSettings)\n'),
    ('app/config.py',
     '        for section in fields(self):\n            getattr(self, section.name).validate()\n        mount = self.storage.required_mount\n        if mount and not self.recordings_dir.is_relative_to(Path(mount)):\n',
     '        for section in fields(self):\n            getattr(self, section.name).validate()\n        if self.motion.analysis_fps > self.camera.framerate:\n            raise ConfigError("motion.analysis_fps cannot be higher than camera.framerate")\n        mount = self.storage.required_mount\n        if mount and not self.recordings_dir.is_relative_to(Path(mount)):\n'),
    ('app/recorder.py',
     'from __future__ import annotations\n\nimport logging\nimport math\n',
     'from __future__ import annotations\n\nimport dataclasses\nimport logging\nimport math\n'),
    ('app/recorder.py',
     '    frames: int\n    clock_synced: bool | None\n\n\n',
     '    frames: int\n    clock_synced: bool | None\n    has_motion: bool | None = None   # None when motion detection is off\n\n\n'),
    ('app/recorder.py',
     '        self._boundary_thread: threading.Thread | None = None\n        self._listeners: list[Callable[[Segment], None]] = []\n        self.last_segment: Segment | None = None\n\n    def add_listener(self, callback: Callable[[Segment], None]) -> None:\n        self._listeners.append(callback)\n\n    @property\n',
     '        self._boundary_thread: threading.Thread | None = None\n        self._listeners: list[Callable[[Segment], None]] = []\n        self._motion = None\n        self._motion_listeners: list[Callable] = []\n        self.last_segment: Segment | None = None\n\n    def add_listener(self, callback: Callable[[Segment], None]) -> None:\n        self._listeners.append(callback)\n\n    def add_motion_listener(self, callback: Callable) -> None:\n        """callback(kind, event) with kind \'start\' or \'end\'."""\n        self._motion_listeners.append(callback)\n\n    @property\n    def motion_active(self) -> bool | None:\n        return self._motion.active if self._motion else None\n\n    @property\n'),
    ('app/recorder.py',
     '        if not os.access(self.recordings_dir, os.W_OK):\n            raise PermissionError(f"recordings directory is not writable: {self.recordings_dir}")\n        self._finalizer.start()\n\n',
     '        if not os.access(self.recordings_dir, os.W_OK):\n            raise PermissionError(f"recordings directory is not writable: {self.recordings_dir}")\n        # Imported before the encoder starts: loading OpenCV holds the interpreter lock long\n        # enough to make the encoder thread drop frames.\n        motion_detector_cls = self._load_motion_detector() if self.settings.motion.enabled else None\n        self._finalizer.start()\n\n'),
    ('app/recorder.py',
     '                 self.recordings_dir, self.settings.recording.segment_seconds, cam.bitrate / 1e6,\n                 cam.keyframe_seconds)\n\n    def stop(self) -> None:\n        was_recording = self._recording\n        self._boundary_stop.set()\n        if self._boundary_thread is not None and self._boundary_thread.is_alive():\n',
     '                 self.recordings_dir, self.settings.recording.segment_seconds, cam.bitrate / 1e6,\n                 cam.keyframe_seconds)\n        if motion_detector_cls is not None:\n            self._motion = motion_detector_cls(self._camera.picam2, self.settings.motion,\n                                               self._motion_listeners)\n            self._motion.start()\n\n    @staticmethod\n    def _load_motion_detector():\n        try:\n            from app.motion_detector import MotionDetector\n        except ImportError as exc:\n            log.error("Motion detection disabled: OpenCV is not installed (%s). "\n                      "Install it with: sudo apt install -y python3-opencv", exc)\n            return None\n        return MotionDetector\n\n    def stop(self) -> None:\n        was_recording = self._recording\n        if self._motion is not None:\n            self._motion.stop()\n        self._boundary_stop.set()\n        if self._boundary_thread is not None and self._boundary_thread.is_alive():\n'),
    ('app/recorder.py',
     '        fsync_directory(seg.final_path.parent)\n        segment = self._output.segment_metadata(seg, seg.final_path.stat().st_size)\n        self.last_segment = segment\n        log.info("Segment completed: %s start=%s duration=%.1fs frames=%d size=%.1fMB%s",\n                 segment.rel_path, segment.start_utc.isoformat(timespec="milliseconds"),\n                 segment.duration_s, segment.frames, segment.size_bytes / 1e6,\n                 {True: "", False: " (clock NOT synchronised)"}.get(segment.clock_synced,\n                                                                     " (clock sync unknown)"))\n',
     '        fsync_directory(seg.final_path.parent)\n        segment = self._output.segment_metadata(seg, seg.final_path.stat().st_size)\n        if self._motion is not None:\n            events = self._motion.events_between(segment.start_utc.timestamp(), segment.end_utc.timestamp())\n            segment = dataclasses.replace(segment, has_motion=bool(events))\n        self.last_segment = segment\n        log.info("Segment completed: %s start=%s duration=%.1fs frames=%d size=%.1fMB%s%s",\n                 segment.rel_path, segment.start_utc.isoformat(timespec="milliseconds"),\n                 segment.duration_s, segment.frames, segment.size_bytes / 1e6,\n                 {True: " motion=yes", False: " motion=no"}.get(segment.has_motion, ""),\n                 {True: "", False: " (clock NOT synchronised)"}.get(segment.clock_synced,\n                                                                     " (clock sync unknown)"))\n'),
    ('app/storage_manager.py',
     '        if not self._warned_capacity and self._max_bytes > used + disk.free - self._min_free_bytes:\n            self._warned_capacity = True\n            log.warning("Maximum storage %.1f GB does not fit on this disk while keeping %.1f GB free "\n                        "(room for about %.1f GB); the free-space limit will apply first",\n                        self._max_bytes / GB, self._min_free_bytes / GB,\n                        max(0, used + disk.free - self._min_free_bytes) / GB)\n',
     '        if not self._warned_capacity and self._max_bytes > used + disk.free - self._min_free_bytes:\n            self._warned_capacity = True\n            log.warning("Maximum storage %g GB does not fit on this disk while keeping %g GB free "\n                        "(room for about %.2f GB); the free-space limit will apply first",\n                        self._max_bytes / GB, self._min_free_bytes / GB,\n                        max(0, used + disk.free - self._min_free_bytes) / GB)\n'),
    ('app/storage_manager.py',
     '        elif disk.free < self._min_free_bytes:\n            state = "low"\n            message = (f"only {disk.free / GB:.2f} GB free (minimum {self._min_free_bytes / GB:g} GB); "\n                       f"something other than recordings is filling the disk")\n        elif used > self._max_bytes:\n            state = "low"\n',
     '        elif disk.free < self._min_free_bytes:\n            state = "low"\n            message = (f"only {disk.free / GB:.2f} GB free (minimum {self._min_free_bytes / GB:g} GB) and no "\n                       f"older recordings left to delete; check what else is using the disk")\n        elif used > self._max_bytes:\n            state = "low"\n'),
    ('tools/phase5_storage_report.py',
     '          f"{fmt_time(complete[-1].start if complete else None)}")\n\n    full = [f for f in complete if f.start % seg_len == 0 and not f.path.name.endswith(RECOVERED_SUFFIX)]\n    sample = full[-BITRATE_SAMPLE_SEGMENTS - 1:-1] or full[-BITRATE_SAMPLE_SEGMENTS:]\n    if sample:\n        bytes_per_s = sum(f.size for f in sample) / (len(sample) * seg_len)\n',
     '          f"{fmt_time(complete[-1].start if complete else None)}")\n\n    # Only segments that filled their whole slot (the next one starts exactly one length later).\n    full = [a for a, b in zip(complete, complete[1:])\n            if a.start % seg_len == 0 and b.start - a.start == seg_len\n            and not a.path.name.endswith(RECOVERED_SUFFIX)]\n    sample = full[-BITRATE_SAMPLE_SEGMENTS:]\n    if sample:\n        bytes_per_s = sum(f.size for f in sample) / (len(sample) * seg_len)\n'),
    ('tools/phase4_check_segments.py',
     '            continue\n        missing = (cur_start - prev_start).total_seconds() - prev_dur\n        # The first segment after a (re)start is named after its real start time, not a boundary.\n        restarted = cur_start.timestamp() % seg_len != 0\n        if restarted:\n            print(f"  [INFO] {prev.name} -> {cur.name}: recorder stopped/restarted, "\n',
     '            continue\n        missing = (cur_start - prev_start).total_seconds() - prev_dur\n        # The first segment after a (re)start is named after its real start time, not a boundary;\n        # a restart that happens to land on a boundary shows up as a gap of more than 10 s.\n        restarted = cur_start.timestamp() % seg_len != 0 or missing > 10\n        if restarted:\n            print(f"  [INFO] {prev.name} -> {cur.name}: recorder stopped/restarted, "\n'),
]

texts = {}
for rel, old, new in EDITS:
    text = texts.setdefault(rel, Path(rel).read_text())
    if text.count(old) != 1:
        sys.exit(f"ABORTED, nothing changed: {rel} does not match the expected 0.5.0 code "
                 f"(found {text.count(old)} matches for:\n{old})")
    texts[rel] = text.replace(old, new)
for rel, text in texts.items():
    shutil.copy2(rel, rel + ".bak")
    Path(rel).write_text(text)
    print(f"updated {rel}  (backup: {rel}.bak)")
print("Update to 0.6.0 complete.")
EOF
python3 update_to_0_6_0.py
```

It should print six `updated …` lines and then `Update to 0.6.0 complete.` If it prints `ABORTED`, nothing was changed; send me the message.

**2. The new motion detector:**

```bash
cat > ~/surveillance/app/motion_detector.py <<'EOF'
"""Lightweight motion detection on the low-resolution stream.

No AI and no object recognition: each analysed frame is compared with a slowly updating
background image, and the largest connected region of changed pixels decides whether
something moved. Frame scores are turned into events with a debounce (movement must be
seen in several frames in a row) and a cooldown (an event only ends after a quiet period),
so one continuous movement produces one event.
"""
from __future__ import annotations

import logging
import threading
import time
from collections import deque
from dataclasses import dataclass, field
from typing import Callable

import cv2
import numpy as np

from app.config import MotionSettings

log = logging.getLogger("MotionDetector")

ANALYSIS_WIDTH = 320
BLUR_KERNEL = (5, 5)
BACKGROUND_ALPHA = 0.04
MOVING_ALPHA_FACTOR = 0.5
RELEARN_ALPHA = 0.5
LIGHTING_CHANGE_PERCENT = 60.0
RELEARN_SECONDS = 1.0
MAX_EVENT_SECONDS = 3600.0
CAPTURE_TIMEOUT_S = 2.0
RECENT_EVENTS = 500


def sensitivity_to_threshold(sensitivity: int) -> int:
    """Sensitivity 1..100 -> minimum brightness change (0..255) for a pixel to count as changed."""
    return round(70 - 0.6 * sensitivity)


@dataclass(frozen=True)
class FrameScore:
    largest_percent: float   # biggest connected changed region, % of the frame
    total_percent: float     # all changed pixels, % of the frame
    moving: bool
    lighting_change: bool


@dataclass
class MotionEvent:
    start: float                     # UTC epoch seconds
    end: float | None = None
    peak_percent: float = 0.0

    @property
    def duration(self) -> float:
        return (self.end if self.end is not None else time.time()) - self.start


class MotionAnalyzer:
    """Pure image analysis: feed greyscale frames, get a score. No camera and no threads."""

    def __init__(self, settings: MotionSettings) -> None:
        self.threshold = sensitivity_to_threshold(settings.sensitivity)
        self.min_area_percent = settings.min_area_percent
        self._background: np.ndarray | None = None
        self._relearn_until = 0.0
        self.last_mask: np.ndarray | None = None

    def analyze(self, gray: np.ndarray, now: float) -> FrameScore:
        blurred = cv2.GaussianBlur(gray, BLUR_KERNEL, 0)
        if self._background is None:
            self._background = blurred.astype(np.float32)
            self._relearn_until = now + RELEARN_SECONDS
            return FrameScore(0.0, 0.0, False, False)
        if now < self._relearn_until:
            cv2.accumulateWeighted(blurred, self._background, RELEARN_ALPHA)
            return FrameScore(0.0, 0.0, False, False)

        diff = cv2.absdiff(blurred, cv2.convertScaleAbs(self._background))
        _, mask = cv2.threshold(diff, self.threshold, 255, cv2.THRESH_BINARY)
        mask = cv2.dilate(mask, None, iterations=2)
        self.last_mask = mask
        # Moving pixels are learned more slowly so a moving object leaves little "ghost" trail
        # behind it; something that stops (a parked car) still becomes background after a while.
        cv2.accumulateWeighted(blurred, self._background, BACKGROUND_ALPHA, mask=cv2.bitwise_not(mask))
        cv2.accumulateWeighted(blurred, self._background, BACKGROUND_ALPHA * MOVING_ALPHA_FACTOR, mask=mask)
        total = cv2.countNonZero(mask) * 100.0 / mask.size
        if total >= LIGHTING_CHANGE_PERCENT:
            # Lights switched on/off or an exposure jump: relearn instead of reporting motion.
            self._background = blurred.astype(np.float32)
            self._relearn_until = now + RELEARN_SECONDS
            return FrameScore(0.0, total, False, True)
        count, _, stats, _ = cv2.connectedComponentsWithStats(mask, connectivity=8)
        largest = float(stats[1:, cv2.CC_STAT_AREA].max()) * 100.0 / mask.size if count > 1 else 0.0
        return FrameScore(largest, total, largest >= self.min_area_percent, False)


class MotionEventTracker:
    """Turns per-frame scores into events: debounce, cooldown, one event per continuous movement."""

    def __init__(self, trigger_frames: int, cooldown_seconds: float,
                 on_start: Callable[[MotionEvent], None], on_end: Callable[[MotionEvent], None]) -> None:
        self._trigger_frames = trigger_frames
        self._cooldown = cooldown_seconds
        self._on_start = on_start
        self._on_end = on_end
        self._streak = 0
        self._streak_start = 0.0
        self._streak_peak = 0.0
        self._last_motion = 0.0
        self.current: MotionEvent | None = None

    def update(self, moving: bool, percent: float, now: float) -> None:
        if moving:
            if self._streak == 0:
                self._streak_start = now
                self._streak_peak = 0.0
            self._streak += 1
            self._streak_peak = max(self._streak_peak, percent)
            self._last_motion = now
            if self.current is None:
                if self._streak >= self._trigger_frames:
                    self.current = MotionEvent(start=self._streak_start, peak_percent=self._streak_peak)
                    self._on_start(self.current)
            else:
                self.current.peak_percent = max(self.current.peak_percent, percent)
                if now - self.current.start >= MAX_EVENT_SECONDS:
                    self._finish(now)
                    self.current = MotionEvent(start=now, peak_percent=percent)
                    self._on_start(self.current)
        else:
            self._streak = 0
            if self.current is not None and now - self._last_motion >= self._cooldown:
                self._finish(self._last_motion)

    def flush(self) -> None:
        if self.current is not None:
            self._finish(self._last_motion)

    def _finish(self, end: float) -> None:
        event, self.current = self.current, None
        event.end = max(end, event.start)
        self._on_end(event)


@dataclass
class _Stats:
    frames: int = 0
    processing_s: float = 0.0
    last_score: FrameScore | None = None
    recent: deque = field(default_factory=lambda: deque(maxlen=RECENT_EVENTS))


class MotionDetector:
    """Background thread that reads the lores stream of a running Picamera2 instance."""

    def __init__(self, picam2, settings: MotionSettings,
                 listeners: list[Callable[[str, MotionEvent], None]] | None = None) -> None:
        self._picam2 = picam2
        self._settings = settings
        self._listeners = listeners or []
        self._analyzer = MotionAnalyzer(settings)
        self._tracker = MotionEventTracker(settings.trigger_frames, settings.cooldown_seconds,
                                           self._started, self._ended)
        self._stats = _Stats()
        self._lock = threading.Lock()
        self._stop = threading.Event()
        self._thread = threading.Thread(target=self._run, name="motion-detector", daemon=True)

    @property
    def active(self) -> bool:
        return self._tracker.current is not None

    @property
    def average_processing_ms(self) -> float:
        s = self._stats
        return s.processing_s / s.frames * 1000 if s.frames else 0.0

    def events_between(self, start: float, end: float) -> list[MotionEvent]:
        """Events (finished or ongoing) that overlap [start, end]."""
        with self._lock:
            events = list(self._stats.recent)
        now = time.time()
        return [e for e in events if e.start <= end and (e.end if e.end is not None else now) >= start]

    def start(self) -> None:
        log.info("Motion detection on: sensitivity %d (pixel threshold %d), min area %.2f%%, "
                 "cooldown %g s, %g analysed frames/s", self._settings.sensitivity,
                 self._analyzer.threshold, self._settings.min_area_percent,
                 self._settings.cooldown_seconds, self._settings.analysis_fps)
        self._thread.start()

    def stop(self) -> None:
        self._stop.set()
        if self._thread.is_alive():
            self._thread.join(timeout=CAPTURE_TIMEOUT_S + 3)
        self._tracker.flush()

    def _started(self, event: MotionEvent) -> None:
        with self._lock:
            self._stats.recent.append(event)
        log.info("Motion started (area %.1f%% of the frame)", event.peak_percent)
        self._notify("start", event)

    def _ended(self, event: MotionEvent) -> None:
        log.info("Motion ended after %.1f s (peak area %.1f%%)", event.end - event.start, event.peak_percent)
        self._notify("end", event)

    def _notify(self, kind: str, event: MotionEvent) -> None:
        for callback in self._listeners:
            try:
                callback(kind, event)
            except Exception:  # noqa: BLE001
                log.exception("Motion listener failed")

    def _run(self) -> None:
        stream = self._picam2.stream_configuration("lores")
        width, height = stream["size"]
        stride = stream["stride"]
        step = max(1, width // ANALYSIS_WIDTH)
        interval = 1.0 / self._settings.analysis_fps
        next_due = time.monotonic()
        while not self._stop.is_set():
            try:
                buffer = self._picam2.capture_buffer("lores", wait=CAPTURE_TIMEOUT_S)
            except TimeoutError:
                continue  # the recorder's health check handles a stalled camera
            except Exception as exc:  # noqa: BLE001
                if self._stop.is_set():
                    return
                log.error("Cannot read the low-resolution stream: %s", exc)
                self._stop.wait(1.0)
                continue
            now = time.time()
            # YUV420: the first stride*height bytes are the Y (greyscale) plane.
            gray = np.ascontiguousarray(buffer[: stride * height].reshape(height, stride)[::step, :width:step])
            started = time.perf_counter()
            score = self._analyzer.analyze(gray, now)
            self._tracker.update(score.moving, score.largest_percent, now)
            self._stats.processing_s += time.perf_counter() - started
            self._stats.frames += 1
            self._stats.last_score = score
            if score.lighting_change:
                log.debug("Lighting change (%.0f%% of pixels changed); background relearned", score.total_percent)

            next_due += interval
            delay = next_due - time.monotonic()
            if delay > 0:
                self._stop.wait(delay)
            else:
                next_due = time.monotonic()
EOF
```

**3. The tuning tool:**

```bash
cat > ~/surveillance/tools/phase6_motion_tune.py <<'EOF'
#!/usr/bin/env python3
"""Phase 6: watch live motion scores to tune sensitivity and minimum area.

Stop the recorder first (only one program can use the camera), then run:

    python3 ~/surveillance/tools/phase6_motion_tune.py [--seconds 60] [--sensitivity N]
                                                      [--min-area PERCENT] [--save-every 5]

Once per second it prints the largest moving region (% of the frame) and whether that
counts as motion. At the end it reports the noise level of a quiet scene, so you can check
that min_area_percent sits safely above it. --save-every writes side-by-side images
(camera view | changed pixels) to ~/surveillance/snapshots/phase6/.
"""
from __future__ import annotations

import argparse
import dataclasses
import os
import sys
import time
from pathlib import Path

os.environ.setdefault("LIBCAMERA_LOG_LEVELS", "*:WARN")
sys.path.insert(0, str(Path(__file__).resolve().parent.parent))

import cv2  # noqa: E402
import numpy as np  # noqa: E402

from app.camera import Camera, CameraError  # noqa: E402
from app.config import DEFAULT_CONFIG_PATH, ConfigError, load_settings  # noqa: E402
from app.motion_detector import ANALYSIS_WIDTH, MotionAnalyzer, MotionEventTracker  # noqa: E402

SNAPSHOT_DIR = Path(__file__).resolve().parent.parent / "snapshots" / "phase6"
BAR_WIDTH = 30


def bar(percent: float, scale: float) -> str:
    filled = min(BAR_WIDTH, int(percent / scale * BAR_WIDTH)) if scale else 0
    return "#" * filled + "." * (BAR_WIDTH - filled)


def percentile(values: list[float], pct: float) -> float:
    if not values:
        return 0.0
    ordered = sorted(values)
    return ordered[min(len(ordered) - 1, int(len(ordered) * pct / 100))]


def main() -> int:
    parser = argparse.ArgumentParser(description="Phase 6: motion tuning")
    parser.add_argument("--config", type=Path, default=DEFAULT_CONFIG_PATH)
    parser.add_argument("--seconds", type=int, default=60)
    parser.add_argument("--sensitivity", type=int, help="override motion.sensitivity (1-100)")
    parser.add_argument("--min-area", type=float, help="override motion.min_area_percent")
    parser.add_argument("--save-every", type=float, default=0, help="save a debug image every N seconds")
    args = parser.parse_args()
    if os.geteuid() == 0:
        print("Do not run this as root.", file=sys.stderr)
        return 2

    try:
        settings, _ = load_settings(args.config, create_if_missing=False)
        motion = settings.motion
        if args.sensitivity is not None:
            motion = dataclasses.replace(motion, sensitivity=args.sensitivity)
        if args.min_area is not None:
            motion = dataclasses.replace(motion, min_area_percent=args.min_area)
        motion.validate()
    except ConfigError as exc:
        print(f"Configuration error: {exc}", file=sys.stderr)
        return 2

    camera = Camera(settings.camera)
    try:
        camera.open()
    except CameraError as exc:
        print(f"{exc}\nIs the recorder still running? Stop it first (Ctrl+C in its window).", file=sys.stderr)
        return 1

    analyzer = MotionAnalyzer(motion)
    events: list[tuple[float, float, float]] = []
    tracker = MotionEventTracker(
        motion.trigger_frames, motion.cooldown_seconds,
        on_start=lambda e: print(f"  >>> MOTION STARTED (area {e.peak_percent:.1f}%)", flush=True),
        on_end=lambda e: (events.append((e.start, e.end, e.peak_percent)),
                          print(f"  <<< motion ended after {e.end - e.start:.1f} s "
                                f"(peak {e.peak_percent:.1f}%)", flush=True)))
    if args.save_every:
        SNAPSHOT_DIR.mkdir(parents=True, exist_ok=True)

    print(f"Sensitivity {motion.sensitivity} (pixel threshold {analyzer.threshold}), "
          f"min area {motion.min_area_percent:g}%, trigger {motion.trigger_frames} frames, "
          f"cooldown {motion.cooldown_seconds:g} s, {motion.analysis_fps:g} frames/s. "
          f"Running {args.seconds} s; Ctrl+C stops.\n")
    quiet_scores: list[float] = []
    processing: list[float] = []
    picam2 = camera.picam2
    try:
        picam2.start()
        stream = picam2.stream_configuration("lores")
        width, height = stream["size"]
        stride = stream["stride"]
        step = max(1, width // ANALYSIS_WIDTH)
        interval = 1.0 / motion.analysis_fps
        end_at = time.monotonic() + args.seconds
        next_print = next_save = time.monotonic()
        second_max = 0.0
        while time.monotonic() < end_at:
            loop_start = time.monotonic()
            buffer = picam2.capture_buffer("lores", wait=2.0)
            now = time.time()
            gray = np.ascontiguousarray(buffer[: stride * height].reshape(height, stride)[::step, :width:step])
            t0 = time.perf_counter()
            score = analyzer.analyze(gray, now)
            tracker.update(score.moving, score.largest_percent, now)
            processing.append((time.perf_counter() - t0) * 1000)
            second_max = max(second_max, score.largest_percent)
            if tracker.current is None and not score.moving:
                quiet_scores.append(score.largest_percent)

            if loop_start >= next_print:
                state = "MOTION" if tracker.current else ("moving" if score.moving else "quiet")
                light = "  (lighting change ignored)" if score.lighting_change else ""
                print(f"{time.strftime('%H:%M:%S')}  largest {second_max:5.2f}%  "
                      f"[{bar(second_max, motion.min_area_percent * 4)}]  {state}{light}", flush=True)
                second_max = 0.0
                next_print = loop_start + 1.0
            if args.save_every and analyzer.last_mask is not None and loop_start >= next_save:
                side = np.hstack([gray, analyzer.last_mask])
                path = SNAPSHOT_DIR / f"{time.strftime('%H%M%S')}_motion.png"
                cv2.imwrite(str(path), side)
                next_save = loop_start + args.save_every
            delay = interval - (time.monotonic() - loop_start)
            if delay > 0:
                time.sleep(delay)
    except KeyboardInterrupt:
        pass
    finally:
        tracker.flush()
        camera.close()

    noise = percentile(quiet_scores, 99)
    print("\n=== Summary ===")
    print(f"Frames analysed      : {len(processing)}, average {np.mean(processing) if processing else 0:.1f} ms "
          f"per frame (max {max(processing) if processing else 0:.1f} ms)")
    print(f"Events               : {len(events)}")
    print(f"Quiet-scene noise    : 99% of quiet frames had a largest region below {noise:.2f}%")
    if noise and motion.min_area_percent < 2 * noise:
        print(f"Advice               : min_area_percent {motion.min_area_percent:g} is close to the noise level; "
              f"try {max(2 * noise, 0.1):.2f} or a lower sensitivity")
    else:
        print(f"Advice               : min_area_percent {motion.min_area_percent:g} is comfortably above the noise")
    if args.save_every:
        print(f"Debug images         : {SNAPSHOT_DIR}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
EOF
```

## Test procedure

Run everything as `ysak` from `~/surveillance`.

**0. Finish Phase 5: the real critical-storage test, then restore the normal settings.**

```bash
python3 tools/set_setting.py storage.min_free_gb 120
python3 -m app.main
```

Within about 2 s of starting, it should log `Storage critical … PAUSED` and then `Recording paused: storage is critically full`, and no `Segment completed` lines should follow. Leave it for about 30 s, then press Ctrl+C.

Then restore the normal settings. Keep 60-second segments for the rest of this phase:

```bash
python3 tools/set_setting.py storage.min_free_gb 2
python3 tools/set_setting.py storage.max_storage_gb 45
python3 tools/set_setting.py motion.sensitivity 70      # also writes the new motion section into the file
```

**1. Tune on your real scene.** The recorder must be stopped for this. Run the tool for 60 s: stay out of view for the first 20 s, walk through the view at about 25 s, then leave again.

```bash
python3 tools/phase6_motion_tune.py --seconds 60 --save-every 5
```

**2. Look at the debug images on your laptop** (run this on the laptop). In each image, the left half is what the detector sees and the right half is the changed pixels. You should see only yourself in white, not flickering noise.

```bash
scp 'ysak@ysak.local:~/surveillance/snapshots/phase6/*.png' .
```

**3. Test room-light changes.** Rerun the tool for 30 s and switch the room light off and on. Expect `(lighting change ignored)`, not `MOTION STARTED`. With your camera's slow auto-exposure, this may show up as a short event instead. Tell me if it does.

**4. Run with the recorder** for about 4 minutes, walking through the view once around the middle:

```bash
python3 -m app.main
```

**5. Measure CPU while it records** (in a second window):

```bash
top -b -n 3 -d 5 -p "$(pgrep -f '^python3 -m app.main')" | grep python3
```

## Expected output

The tuning tool (step 1) should look roughly like this:

```text
Sensitivity 70 (pixel threshold 28), min area 0.5%, trigger 3 frames, cooldown 10 s, 5 frames/s. Running 60 s; Ctrl+C stops.

22:10:01  largest  0.00%  [..............................]  quiet
...
22:10:25  largest  6.84%  [##############################]  moving
  >>> MOTION STARTED (area 4.1%)
22:10:26  largest 11.20%  [##############################]  MOTION
...
  <<< motion ended after 8.4 s (peak 14.9%)

=== Summary ===
Frames analysed      : 300, average 3.0 ms per frame (max 8.0 ms)
Events               : 1
Quiet-scene noise    : 99% of quiet frames had a largest region below 0.05%
Advice               : min_area_percent 0.5 is comfortably above the noise
```

The recorder (step 4) should look roughly like this:

```text
INFO Recorder: Started recording to ... (60 s segments, 2.50 Mbit/s, keyframe every 2 s)
INFO MotionDetector: Motion detection on: sensitivity 70 (pixel threshold 28), min area 0.50%, cooldown 10 s, 5 analysed frames/s
INFO Recorder: Segment completed: 2026-09-24/00-11-00Z.mp4 ... size=9.4MB motion=no
INFO MotionDetector: Motion started (area 3.9% of the frame)
INFO MotionDetector: Motion ended after 9.2 s (peak area 15.3%)
INFO Recorder: Segment completed: 2026-09-24/00-12-00Z.mp4 ... size=9.6MB motion=yes
```

For CPU, I expect roughly 12–15 % of one core, about the same as Phase 3. The detector itself should add only about 2 %.

## Tuning guide

| Problem | Change |
|---|---|
| Events with nobody there (noise, flicker) | Lower the sensitivity (e.g. 55), or raise `min_area_percent` above twice the "quiet-scene noise" figure |
| A small or distant person is missed | Raise the sensitivity (e.g. 80) or lower `min_area_percent` (e.g. 0.2) |
| One walk-through is split into several events | Raise `cooldown_seconds` (e.g. 20) |
| A short blip starts an event | Raise `trigger_frames` (e.g. 5) |
| Lots of events at night | Expected with this sensor: it has no infrared, so the image is mostly noise in the dark. Use a lower night sensitivity, or add an IR light (Phase 14 performance guide) |

For example: `python3 tools/set_setting.py motion.sensitivity 55`. You can also try values temporarily with the tool first: `--sensitivity 55 --min-area 1.0`.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `ABORTED, nothing changed` from the update script | One of your files differs from what I sent. Send me the message and I'll give you full files. |
| `Motion detection disabled: OpenCV is not installed` | `sudo apt install -y python3-opencv` |
| The tuning tool says the camera is in use | Stop the recorder first (Ctrl+C in its window). |
| Frame gaps return in the Phase 4 checker | Send me the checker output. OpenCV is loaded before recording starts, so this shouldn't happen. |

**Please send me:**
- the result of step 0 (the critical pause);
- the tuning tool's summary from steps 1 and 3;
- the recorder's log from step 4, showing the motion lines and `motion=yes/no`;
- the CPU figure from step 5.

Phase 7 then stores recordings and motion events in SQLite, the index the web interface will use.
