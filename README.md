Phase 3 (hardware H.264 video recording) is ready for you to run on the Pi. I tested it here with Picamera2 0.3.37's real MP4-writing code fed genuine H.264 frames, since there's no camera in this environment. Every check passed, including a simulated power cut.

Since you said the Phase 2 output was fine, I'm assuming the picture is the right way up and the colours look normal. I've set the recording default to the **wide** 1296×972 mode (the whole field of view). If the image turns out upside down, add `--rotate180` to the command. Your folder listing also shows 20:05 local time against 12:05 in the filename, so your timezone is UTC+8. Naming files in UTC is working as intended.

## Design changes from what I found this phase

- **The separate ffmpeg recording process is gone.** Picamera2 0.3.37, the version you have, has two output classes that do the job better:
  - `PyavOutput` writes MP4 files inside our own program, using the camera's exact frame timestamps. This fixes the timestamp-drift risk I listed in the design.
  - `SplittableOutput` switches to a new file at the next keyframe without dropping frames. That is exactly what Phase 4's 5-minute segments need.
- **Fragmented MP4 is confirmed as the right format.** When I cut a file to 70 % of its size to simulate a power cut, the fragmented copy was still playable (8.2 s of 12 s). The normal MP4 was unreadable.
- **Browser seeking needs one specific thing from the web server.** In Chrome, the fragmented MP4 showed the full 12.04 s length and seeked correctly, but only when the web server supports byte-range requests. With a simple server that doesn't, Chrome showed a 2-second video and couldn't seek. The Phase 9 web app will support range requests; for now you can just open the files directly.

I've updated `docs/DESIGN.md` to match: new defaults of 1296×972 recording, a 640×480 low-resolution stream, and 320×240 motion analysis. At 2.5 Mbit/s, 45 GB holds about 40 hours of recording.

---

# Phase 3: hardware H.264 video recording

## What we're building

A test that records one video from the Pi 4's hardware H.264 encoder. The same stream is written to two files at once:

| File | Format | Why |
|---|---|---|
| `…_plain.mp4` | Normal MP4 | For comparison |
| `…_fragmented.mp4` | Fragmented MP4, in 2-second self-contained pieces | The format 24/7 recording will use |

**Settings:** 1296×972 at 15 fps and 2.5 Mbit/s, with a keyframe every 2 s. The 640×480 low-resolution stream is also turned on, so the camera runs exactly as it will in the final system.

**How the files are saved:** each file is written as `*.mp4.partial`, flushed to disk and only then renamed to `.mp4`. So a finished file is never half-written.

**What the script checks afterwards:**
- the codec and resolution;
- frame count and frame rate;
- **dropped frames**, found by looking for gaps between frame timestamps;
- the keyframe interval (Phase 4's file splitting depends on it);
- the measured bitrate, converted into hours per 45 GB;
- how much CPU the recording used;
- a **simulated power cut** on truncated copies of both files.

## Install

```bash
sudo apt install -y python3-av ffmpeg
```

`python3-av` provides PyAV, which Picamera2 uses to write MP4 files. `ffmpeg` is already installed, and it includes `ffprobe`, which the script uses for checking.

## File to create

The file goes at `/home/ysak/surveillance/tools/phase3_record_video.py`. Create it as `ysak`:

```bash
nano ~/surveillance/tools/phase3_record_video.py
```

```python
#!/usr/bin/env python3
"""Phase 3: record H.264 video with the Pi 4 hardware encoder.

Run on the Raspberry Pi as your normal (non-root) user:

    python3 ~/surveillance/tools/phase3_record_video.py

One hardware-encoded stream is written to two files at the same time:
  <stamp>_plain.mp4        normal MP4 (index written at the end)
  <stamp>_fragmented.mp4   fragmented MP4 (self-contained ~2 s fragments)

Afterwards the script checks both files with ffprobe (codec, resolution, frame count,
timestamp gaps, keyframe interval, bitrate), and simulates a power cut by probing
truncated copies of each file. Exit code 0 = recording OK.
"""
from __future__ import annotations

import argparse
import json
import os
import shutil
import subprocess
import sys
import time
from datetime import datetime, timezone
from pathlib import Path
from typing import BinaryIO, Callable

os.environ.setdefault("LIBCAMERA_LOG_LEVELS", "*:WARN")

DEFAULT_OUTPUT = Path.home() / "surveillance" / "test-recordings" / "phase3"
FRAGMENTED_MOVFLAGS = "frag_keyframe+empty_moov+default_base_moof"
TRUNCATE_FRACTION = 0.7


class Report:
    def __init__(self) -> None:
        self.counts = {"PASS": 0, "WARN": 0, "FAIL": 0, "INFO": 0}

    def _emit(self, status: str, name: str, detail: str = "", hint: str = "") -> None:
        self.counts[status] += 1
        print(f"[{status}] {name}" + (f": {detail}" if detail else ""), flush=True)
        for hint_line in hint.splitlines():
            print(f"       -> {hint_line}", flush=True)

    def ok(self, name: str, detail: str = "") -> None:
        self._emit("PASS", name, detail)

    def warn(self, name: str, detail: str = "", hint: str = "") -> None:
        self._emit("WARN", name, detail, hint)

    def fail(self, name: str, detail: str = "", hint: str = "") -> None:
        self._emit("FAIL", name, detail, hint)

    def info(self, name: str, detail: str = "") -> None:
        self._emit("INFO", name, detail)


def parse_size(value: str) -> tuple[int, int]:
    try:
        width, height = (int(part) for part in value.lower().split("x"))
    except ValueError:
        raise argparse.ArgumentTypeError(f"expected WIDTHxHEIGHT, got {value!r}") from None
    if not (16 <= width <= 1920 and 16 <= height <= 1920):
        raise argparse.ArgumentTypeError(f"size out of range (max 1920): {value!r}")
    return width, height


def fsync_directory(directory: Path) -> None:
    fd = os.open(directory, os.O_RDONLY | os.O_DIRECTORY)
    try:
        os.fsync(fd)
    finally:
        os.close(fd)


def atomic_write(path: Path, writer: Callable[[BinaryIO], None]) -> None:
    tmp = path.with_name(f".{path.name}.tmp")
    try:
        with open(tmp, "wb") as handle:
            writer(handle)
            handle.flush()
            os.fsync(handle.fileno())
        os.replace(tmp, path)
    except BaseException:
        tmp.unlink(missing_ok=True)
        raise
    fsync_directory(path.parent)


def finalize(partial: Path, final: Path) -> None:
    """Flush a completed recording to disk, then atomically give it its final name."""
    with open(partial, "rb+") as handle:
        os.fsync(handle.fileno())
    os.replace(partial, final)
    fsync_directory(final.parent)


def clock_synchronized() -> bool | None:
    try:
        result = subprocess.run(["timedatectl", "show", "-p", "NTPSynchronized", "--value"],
                                capture_output=True, text=True, timeout=5, check=False)
    except (OSError, subprocess.TimeoutExpired):
        return None
    value = result.stdout.strip()
    return {"yes": True, "no": False}.get(value)


def cpu_temperature() -> float | None:
    try:
        return int(Path("/sys/class/thermal/thermal_zone0/temp").read_text()) / 1000
    except (OSError, ValueError):
        return None


def ffprobe_packets(path: Path) -> list[tuple[float, bool]] | None:
    """Return (pts_seconds, is_keyframe) for every video packet, or None if unreadable."""
    try:
        result = subprocess.run(
            ["ffprobe", "-v", "error", "-select_streams", "v:0",
             "-show_entries", "packet=pts_time,flags", "-of", "csv=p=0", str(path)],
            capture_output=True, text=True, timeout=120, check=False)
    except (OSError, subprocess.TimeoutExpired):
        return None
    packets = []
    for line in result.stdout.splitlines():
        pts, _, flags = line.partition(",")
        try:
            packets.append((float(pts), "K" in flags))
        except ValueError:
            continue
    if result.returncode != 0 and not packets:
        return None
    packets.sort()
    return packets


def ffprobe_stream(path: Path) -> dict:
    result = subprocess.run(
        ["ffprobe", "-v", "error", "-select_streams", "v:0",
         "-show_entries", "stream=codec_name,profile,width,height", "-of", "json", str(path)],
        capture_output=True, text=True, timeout=60, check=False)
    try:
        streams = json.loads(result.stdout or "{}").get("streams", [])
    except json.JSONDecodeError:
        streams = []
    return streams[0] if streams else {}


def analyse(path: Path, fps: float) -> dict:
    packets = ffprobe_packets(path) or []
    stream = ffprobe_stream(path)
    size_bytes = path.stat().st_size
    frame_interval = 1.0 / fps
    result = {
        "file": path.name,
        "size_bytes": size_bytes,
        "codec": stream.get("codec_name"),
        "profile": stream.get("profile"),
        "width": stream.get("width"),
        "height": stream.get("height"),
        "frames": len(packets),
    }
    if len(packets) < 2:
        return result
    times = [pts for pts, _ in packets]
    deltas = [b - a for a, b in zip(times, times[1:])]
    keyframes = [pts for pts, key in packets if key]
    duration = times[-1] - times[0] + frame_interval
    result.update({
        "duration_s": round(duration, 3),
        "measured_fps": round((len(times) - 1) / (times[-1] - times[0]), 2),
        "max_gap_ms": round(max(deltas) * 1000, 1),
        "gaps_over_1_5_frames": sum(1 for d in deltas if d > 1.5 * frame_interval),
        "keyframes": len(keyframes),
        "keyframe_interval_s": (round((keyframes[-1] - keyframes[0]) / (len(keyframes) - 1), 2)
                                if len(keyframes) > 1 else None),
        "bitrate_kbps": round(size_bytes * 8 / duration / 1000, 1),
    })
    return result


def truncation_test(path: Path, fraction: float) -> float | None:
    """Copy the first `fraction` of the file and return how many seconds are still playable."""
    probe_copy = path.with_name(f".{path.stem}.truncated{path.suffix}")
    keep = int(path.stat().st_size * fraction)
    try:
        with open(path, "rb") as src, open(probe_copy, "wb") as dst:
            remaining = keep
            while remaining > 0:
                chunk = src.read(min(remaining, 1 << 20))
                if not chunk:
                    break
                dst.write(chunk)
                remaining -= len(chunk)
        packets = ffprobe_packets(probe_copy)
    finally:
        probe_copy.unlink(missing_ok=True)
    if not packets or len(packets) < 2:
        return None
    return packets[-1][0] - packets[0][0]


def pick_sensor_size(sensor_modes: list[dict], size: tuple[int, int]) -> tuple[int, int] | None:
    for mode in sensor_modes:
        if tuple(mode["size"]) == size:
            return size
    return None


def record(args: argparse.Namespace, stamp: str, report: Report) -> dict | None:
    from libcamera import Transform  # noqa: PLC0415
    from picamera2 import Picamera2  # noqa: PLC0415
    from picamera2.encoders import H264Encoder  # noqa: PLC0415
    from picamera2.outputs import PyavOutput  # noqa: PLC0415

    plain_final = args.output / f"{stamp}_plain.mp4"
    frag_final = args.output / f"{stamp}_fragmented.mp4"
    plain_partial = plain_final.with_name(plain_final.name + ".partial")
    frag_partial = frag_final.with_name(frag_final.name + ".partial")

    try:
        picam2 = Picamera2(args.camera)
    except (RuntimeError, IndexError) as exc:
        report.fail("Open camera", str(exc),
                    "Is another program using it? Check: pgrep -af 'rpicam|libcamera|picamera2'")
        return None

    output_errors: list[str] = []
    started_utc = ended_utc = None
    frames_encoded = 0
    cpu_seconds = wall_seconds = 0.0
    interrupted = False
    try:
        sensor_size = pick_sensor_size(picam2.sensor_modes, args.size)
        config = picam2.create_video_configuration(
            main={"size": args.size, "format": "YUV420"},
            lores={"size": args.lores, "format": "YUV420"},
            sensor={"output_size": sensor_size} if sensor_size else {},
            controls={"FrameRate": args.fps},
            transform=Transform(hflip=args.rotate180, vflip=args.rotate180),
        )
        picam2.configure(config)
        report.ok("Configure camera",
                  f"main {args.size[0]}x{args.size[1]} + lores {args.lores[0]}x{args.lores[1]} "
                  f"@ {args.fps:g} fps, sensor mode {sensor_size or 'auto'}")

        encoder = H264Encoder(bitrate=args.bitrate, iperiod=args.keyframe_frames,
                              framerate=args.fps, profile="high", repeat=True)
        plain = PyavOutput(str(plain_partial), format="mp4")
        fragmented = PyavOutput(str(frag_partial), format="mp4",
                                options={"movflags": FRAGMENTED_MOVFLAGS})
        for out in (plain, fragmented):
            out.error_callback = lambda exc: output_errors.append(f"{type(exc).__name__}: {exc}")

        print(f"\nRecording {args.seconds} s to {args.output} (Ctrl+C stops early) ...", flush=True)
        cpu_start = sum(os.times()[:2])
        wall_start = time.monotonic()
        started_utc = datetime.now(timezone.utc)
        picam2.start_recording(encoder, [plain, fragmented])
        try:
            next_progress = 10
            while (elapsed := time.monotonic() - wall_start) < args.seconds:
                time.sleep(0.5)
                if elapsed >= next_progress:
                    size_kib = plain_partial.stat().st_size / 1024 if plain_partial.exists() else 0
                    print(f"  {int(elapsed):>4} s  frames={encoder.frames_encoded}  "
                          f"plain={size_kib:.0f} KiB", flush=True)
                    next_progress += 10
                if output_errors:
                    break
        except KeyboardInterrupt:
            interrupted = True
            print("\nStopping early ...", flush=True)
        finally:
            frames_encoded = encoder.frames_encoded
            picam2.stop_recording()
            ended_utc = datetime.now(timezone.utc)
            wall_seconds = time.monotonic() - wall_start
            cpu_seconds = sum(os.times()[:2]) - cpu_start
    finally:
        picam2.close()

    if output_errors:
        report.fail("Muxer", "; ".join(output_errors))
        return None

    for partial, final in ((plain_partial, plain_final), (frag_partial, frag_final)):
        if not partial.exists() or partial.stat().st_size == 0:
            report.fail("Output file", f"{partial.name} is missing or empty")
            return None
        finalize(partial, final)

    return {
        "plain": plain_final,
        "fragmented": frag_final,
        "started_utc": started_utc.isoformat(timespec="milliseconds"),
        "ended_utc": ended_utc.isoformat(timespec="milliseconds"),
        "frames_encoded": frames_encoded,
        "wall_seconds": round(wall_seconds, 2),
        "cpu_percent_of_one_core": round(cpu_seconds / wall_seconds * 100, 1) if wall_seconds else None,
        "interrupted": interrupted,
    }


def evaluate(report: Report, args: argparse.Namespace, rec: dict) -> dict:
    print("\n=== Checking files with ffprobe ===", flush=True)
    results = {}
    expected_frames = rec["wall_seconds"] * args.fps
    for kind in ("plain", "fragmented"):
        info = analyse(rec[kind], args.fps)
        results[kind] = info
        label = f"{kind} MP4"
        if info["codec"] != "h264" or info["frames"] < 2:
            report.fail(label, f"unreadable or not H.264: {info}")
            continue
        if (info["width"], info["height"]) != args.size:
            report.fail(label, f"resolution {info['width']}x{info['height']}, expected "
                               f"{args.size[0]}x{args.size[1]}")
            continue
        report.ok(label, f"{info['file']}: H.264 {info['profile']}, {info['width']}x{info['height']}, "
                         f"{info['duration_s']:.1f} s, {info['frames']} frames, "
                         f"{info['size_bytes'] / 1e6:.1f} MB")

    plain = results["plain"]
    if plain.get("frames", 0) < 2:
        return results

    frame_ratio = plain["frames"] / expected_frames if expected_frames else 0
    detail = (f"{plain['frames']} frames in {rec['wall_seconds']:.1f} s = {plain['measured_fps']} fps "
              f"(expected ~{expected_frames:.0f})")
    if frame_ratio >= 0.97:
        report.ok("Frame count", detail)
    else:
        report.warn("Frame count", detail, "Frames were dropped. Send me this output.")

    if plain["gaps_over_1_5_frames"] == 0:
        report.ok("Timestamps", f"no gaps (largest step {plain['max_gap_ms']} ms)")
    else:
        report.warn("Timestamps", f"{plain['gaps_over_1_5_frames']} gaps, largest {plain['max_gap_ms']} ms",
                    "Gaps mean frames were dropped between camera and encoder.")

    expected_kf = args.keyframe_frames / args.fps
    kf = plain["keyframe_interval_s"]
    if kf is not None and abs(kf - expected_kf) <= 0.25 * expected_kf:
        report.ok("Keyframe interval", f"{kf} s (requested {expected_kf:g} s)")
    else:
        report.warn("Keyframe interval", f"{kf} s (requested {expected_kf:g} s)",
                    "Segment splitting in Phase 4 depends on regular keyframes.")

    report.info("Bitrate", f"measured {plain['bitrate_kbps']:.0f} kbit/s, target {args.bitrate / 1000:.0f} "
                           f"kbit/s -> {plain['bitrate_kbps'] * 0.45 / 1000:.2f} GB/hour, "
                           f"45 GB = {45 / (plain['bitrate_kbps'] * 0.45 / 1000):.0f} hours")
    report.info("CPU", f"{rec['cpu_percent_of_one_core']} % of one core for the whole recording process")

    print(f"\n=== Power-cut simulation (keep first {TRUNCATE_FRACTION:.0%} of each file) ===", flush=True)
    for kind in ("plain", "fragmented"):
        playable = truncation_test(rec[kind], TRUNCATE_FRACTION)
        results[kind]["playable_after_truncation_s"] = round(playable, 1) if playable else None
        if kind == "plain":
            if playable is None:
                report.info("plain MP4 truncated", "unplayable (expected: its index is written at the end)")
            else:
                report.info("plain MP4 truncated", f"{playable:.1f} s still readable")
        elif playable is not None and playable >= 0.5 * TRUNCATE_FRACTION * plain["duration_s"]:
            report.ok("fragmented MP4 truncated", f"{playable:.1f} s still playable")
        else:
            report.warn("fragmented MP4 truncated", f"playable: {playable}",
                        "Fragmented MP4 did not survive truncation. Send me this output.")
    return results


def build_arg_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(description="Phase 3: hardware H.264 recording test.")
    parser.add_argument("--camera", type=int, default=0)
    parser.add_argument("--seconds", type=int, default=60, help="recording length (default 60)")
    parser.add_argument("--size", type=parse_size, default=(1296, 972),
                        help="recording size (default 1296x972, full field of view on OV5647)")
    parser.add_argument("--lores", type=parse_size, default=(640, 480),
                        help="low-resolution stream size (default 640x480)")
    parser.add_argument("--fps", type=float, default=15.0, help="frame rate (default 15)")
    parser.add_argument("--bitrate", type=int, default=2_500_000, help="bits per second (default 2500000)")
    parser.add_argument("--keyframe-seconds", type=float, default=2.0,
                        help="seconds between keyframes (default 2)")
    parser.add_argument("--rotate180", action="store_true", help="rotate the image 180 degrees")
    parser.add_argument("--output", type=Path, default=DEFAULT_OUTPUT)
    return parser


def main() -> int:
    args = build_arg_parser().parse_args()
    if os.geteuid() == 0:
        print("Do not run this as root. Type 'exit' to leave the root shell.", file=sys.stderr)
        return 2
    if not (5 <= args.seconds <= 3600 and 1 <= args.fps <= 30
            and 250_000 <= args.bitrate <= 17_000_000 and 0.5 <= args.keyframe_seconds <= 10):
        print("Allowed: --seconds 5-3600, --fps 1-30, --bitrate 250000-17000000, "
              "--keyframe-seconds 0.5-10", file=sys.stderr)
        return 2
    if args.lores[0] > args.size[0] or args.lores[1] > args.size[1]:
        print("--lores must not be larger than --size", file=sys.stderr)
        return 2
    args.keyframe_frames = max(1, round(args.keyframe_seconds * args.fps))

    missing = [tool for tool in ("ffprobe",) if shutil.which(tool) is None]
    try:
        import av  # noqa: F401, PLC0415
    except ImportError:
        missing.append("python3-av")
    if missing:
        print(f"Missing: {', '.join(missing)}\nInstall with: sudo apt install -y ffmpeg python3-av",
              file=sys.stderr)
        return 1

    args.output = args.output.expanduser().resolve()
    args.output.mkdir(parents=True, exist_ok=True)
    needed = 2 * args.seconds * args.bitrate / 8 * 1.5 + 50e6
    free = shutil.disk_usage(args.output).free
    if free < needed:
        print(f"Not enough free space: need {needed / 1e6:.0f} MB, have {free / 1e6:.0f} MB", file=sys.stderr)
        return 1

    report = Report()
    stamp = datetime.now(timezone.utc).strftime("%Y%m%dT%H%M%SZ")
    synced = clock_synchronized()
    if synced:
        report.ok("System clock", "NTP synchronised")
    else:
        report.warn("System clock", f"NTP synchronised = {synced}",
                    "Timestamps may be wrong. The Pi 4 has no battery-backed clock.")
    temp_before = cpu_temperature()

    rec = record(args, stamp, report)
    if rec is None:
        print("\nRESULT: RECORDING FAILED")
        return 1
    results = evaluate(report, args, rec)
    temp_after = cpu_temperature()
    if temp_before is not None and temp_after is not None:
        report.info("CPU temperature", f"{temp_before:.1f} C before, {temp_after:.1f} C after")

    summary = {
        "settings": {"size": list(args.size), "lores": list(args.lores), "fps": args.fps,
                     "bitrate": args.bitrate, "keyframe_frames": args.keyframe_frames,
                     "rotate180": args.rotate180},
        "clock_synchronized": synced,
        **{key: value for key, value in rec.items() if key not in ("plain", "fragmented")},
        "files": results,
        "cpu_temperature_c": {"before": temp_before, "after": temp_after},
    }
    summary_path = args.output / f"{stamp}_recording.json"
    atomic_write(summary_path, lambda handle: handle.write(json.dumps(summary, indent=2).encode()))

    counts = report.counts
    print(f"\nMetadata: {summary_path}")
    print(f"PASS={counts['PASS']} WARN={counts['WARN']} FAIL={counts['FAIL']} INFO={counts['INFO']}")
    if counts["FAIL"]:
        print("RESULT: RECORDING FAILED")
        return 1
    print("RESULT: RECORDING OK - now do the playback checks in the instructions.")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

## Test procedure

**1. Record 60 seconds.** While it records, walk or wave in front of the camera for part of the time, so the video contains motion (motion raises the bitrate):

```bash
whoami     # must print: ysak
python3 ~/surveillance/tools/phase3_record_video.py
```

**2. Check playback on your laptop.** Copy the files over by running this on the laptop:

```bash
scp 'ysak@ysak.local:~/surveillance/test-recordings/phase3/*.mp4' .
```

Open **both** files in **VLC** and by dragging each into a **Chrome or Firefox tab**. For each file, check:
- it plays with smooth motion;
- it lasts about 60 s;
- you can drag the seek bar to the middle and it jumps there.

**3. Optional:** run a 10-minute recording to check stability and heat:

```bash
python3 ~/surveillance/tools/phase3_record_video.py --seconds 600
```

## Expected output

```text
[PASS] System clock: NTP synchronised
[PASS] Configure camera: main 1296x972 + lores 640x480 @ 15 fps, sensor mode (1296, 972)

Recording 60 s to /home/ysak/surveillance/test-recordings/phase3 (Ctrl+C stops early) ...
    10 s  frames=150  plain=3000 KiB
    ...
=== Checking files with ffprobe ===
[PASS] plain MP4: …_plain.mp4: H.264 High, 1296x972, 60.0 s, 900 frames, 18.8 MB
[PASS] fragmented MP4: …_fragmented.mp4: H.264 High, 1296x972, 60.0 s, 900 frames, 18.8 MB
[PASS] Frame count: 900 frames in 60.0 s = 15.0 fps (expected ~900)
[PASS] Timestamps: no gaps (largest step 66.7 ms)
[PASS] Keyframe interval: 2.0 s (requested 2 s)
[INFO] Bitrate: measured ~2500 kbit/s ... 45 GB = ~40 hours
[INFO] CPU: a few % of one core for the whole recording process

=== Power-cut simulation (keep first 70% of each file) ===
[INFO] plain MP4 truncated: unplayable (expected: its index is written at the end)
[PASS] fragmented MP4 truncated: ~41 s still playable
...
RESULT: RECORDING OK - now do the playback checks in the instructions.
```

About 66.7 ms between frames is exactly 1/15 s. In a dark room the bitrate may come out below the target, and the CPU figure is the main thing I want to see from real hardware.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `Missing: python3-av` | `sudo apt install -y python3-av` |
| `Muxer` FAIL | Send me the full error text. |
| Frame count or timestamp WARN | Frames were dropped. Send me the output and the result of `vcgencmd get_throttled`. |
| Video is upside down | Run again with `--rotate180`. That setting will carry into the Phase 4 configuration. |
| Chrome/Firefox won't play the file but VLC does | Tell me which browser and which file. That is exactly the compatibility issue this phase is checking for. |

**Please send me:**
- the script's full output;
- whether both files played and seeked in VLC and in your browser;
- whether the image needed `--rotate180`.

Phase 4 turns this into continuous 24/7 recording in 5-minute segments. It starts the real `app/` modules: `config.py`, `logging_setup.py`, `camera.py` and `recorder.py`.
