Phase 1 passed on your Pi, and Phase 2 (still-image capture) is ready for you to run. I tested it here against a fake camera: all four captures saved and re-opened correctly, and a deliberately bad setting made one capture fail without stopping the others. It has not run on real hardware yet.

## What your Phase 1 output tells us

**Healthy:**
- Pi 4 on Trixie, 64-bit, Python 3.13.5.
- Picamera2 0.3.37 and libcamera 0.7.2.
- Both hardware encoders (H.264 and JPEG) are present.
- FFmpeg 7.1.5.
- No under-voltage or throttling, and 49.7 °C.

**Four things to act on:**

1. **Stop working as root.** Your prompt shows a root shell (`root@ysak`). Type `exit` until `whoami` prints `ysak`. The Phase 1 file was probably created as root, so give it back to your user:

```bash
sudo chown -R ysak:ysak /home/ysak/surveillance
```

   The Phase 2 script refuses to run as root, so image files don't end up owned by root.

2. **Your OV5647 (Camera Module v1 type) changes the recording resolution choice.** Its 1920×1080 mode is a **centre crop**: it uses only 1928×1080 of the 2592×1944 sensor, about 41 % of the scene. Its 1296×972 mode sees the **whole field of view** and combines 2×2 pixels, which also helps in low light.
   - For surveillance, a wider view usually beats more pixels on a smaller area, so I'll probably change the design default to **1296×972 at 15 fps**.
   - The motion stream becomes 640×480 (4:3).
   - That needs about 2–2.5 Mbit/s, so 45 GB would last roughly 40–50 hours instead of about 33.
   - Phase 2 saves both views so you can decide.

3. **The room was very dark.** Light level was about 3 lux and exposure was 66.5 ms, which is the entire frame time at 15 fps. Gain was 8.0 and brightness only 26 out of 255. That's fine for a test, but at night this sensor needs infrared light. Does your camera board have IR LEDs, or do daytime pictures look pinkish or purple? If so, it's a NoIR version and needs a different tuning file (I'll handle that in Phase 3).

4. **Minor, for Phase 12:** the hostname `ysak` will show up in public certificate logs once HTTPS is set up, so consider a neutral name like `cam01` before then.

---

# Phase 2: capture a still image

## What we're building

A script that takes four pictures one after another:

| File | What it shows | Why |
|---|---|---|
| `…_full.jpg` | Full 2592×1944 sensor still | Checks focus, lens and orientation |
| `…_wide.jpg` | 1296×972, full field of view | Candidate recording mode |
| `…_crop1080.jpg` | 1920×1080, centre crop | Candidate recording mode, for comparison |
| `…_motion_view.png` | 640×480 greyscale | Exactly what the motion detector will see |

It also writes a `…_captures.json` file with the exposure, gain, light level and file sizes.

It uses **atomic writes**: write to a hidden temporary file, flush it to disk (`fsync`), then rename it over the final name. The recorder and the settings code will use the same method later, so a power cut can never leave a half-written file. It also reads the image picture and exposure data from the same frame (`capture_request`), so they always match.

## Install

This is most likely already installed as a Picamera2 dependency:

```bash
sudo apt install -y python3-pil
```

## File to create

The file is `/home/ysak/surveillance/tools/phase2_still_capture.py`. Create it **as `ysak`**, not root:

```bash
nano ~/surveillance/tools/phase2_still_capture.py
```

```python
#!/usr/bin/env python3
"""Phase 2: capture still images with Picamera2 and save them safely.

Run on the Raspberry Pi as your normal (non-root) user:

    python3 ~/surveillance/tools/phase2_still_capture.py

It captures, one after another:
  full          full sensor resolution still (JPEG)
  wide          full field of view, 2x2 binned sensor mode (JPEG)
  crop1080      1920x1080 sensor mode, which is a centre crop on some sensors (JPEG)
  motion_view   the greyscale low-resolution frame the motion detector will analyse (PNG)

Every file is written atomically (temporary file, fsync, rename), and a JSON file
records the capture metadata. Exit code 0 = all captures succeeded.
"""
from __future__ import annotations

import argparse
import json
import os
import sys
import time
from dataclasses import dataclass
from datetime import datetime, timezone
from pathlib import Path
from typing import BinaryIO, Callable

os.environ.setdefault("LIBCAMERA_LOG_LEVELS", "*:WARN")

DEFAULT_OUTPUT = Path.home() / "surveillance" / "snapshots" / "phase2"


@dataclass(frozen=True)
class CapturePlan:
    name: str
    description: str
    kind: str  # "still", "video" or "motion"
    main_size: tuple[int, int] | None
    sensor_size: tuple[int, int] | None
    lores_size: tuple[int, int] | None = None


def parse_size(value: str) -> tuple[int, int]:
    try:
        width, height = (int(part) for part in value.lower().split("x"))
    except ValueError:
        raise argparse.ArgumentTypeError(f"expected WIDTHxHEIGHT, got {value!r}") from None
    if not (16 <= width <= 4096 and 16 <= height <= 4096):
        raise argparse.ArgumentTypeError(f"size out of range: {value!r}")
    return width, height


def fsync_directory(directory: Path) -> None:
    fd = os.open(directory, os.O_RDONLY | os.O_DIRECTORY)
    try:
        os.fsync(fd)
    finally:
        os.close(fd)


def atomic_write(path: Path, writer: Callable[[BinaryIO], None]) -> int:
    """Write via a hidden temp file in the same directory, then rename over the target.

    A crash or power cut leaves either the old file or the complete new file, never
    a half-written one.
    """
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
    return path.stat().st_size


def build_plans(sensor_modes: list[dict], lores_size: tuple[int, int]) -> list[CapturePlan]:
    sizes = sorted({tuple(mode["size"]) for mode in sensor_modes}, key=lambda s: s[0] * s[1])
    full = sizes[-1]
    full_aspect = full[0] / full[1]
    # Smallest-area mode above 1 MP with the full sensor's aspect ratio: the binned full-FOV mode.
    wide = next((s for s in sizes if s[0] * s[1] >= 1_000_000
                 and abs(s[0] / s[1] - full_aspect) < 0.02 and s != full), full)

    plans = [
        CapturePlan("full", f"full sensor still {full[0]}x{full[1]}", "still", full, full),
        CapturePlan("wide", f"full field of view {wide[0]}x{wide[1]} (binned)", "video", wide, wide),
    ]
    if (1920, 1080) in sizes:
        plans.append(CapturePlan("crop1080", "1920x1080 sensor mode", "video",
                                 (1920, 1080), (1920, 1080)))
    plans.append(CapturePlan("motion_view", f"motion detector view {lores_size[0]}x{lores_size[1]}",
                             "motion", wide, wide, lores_size))
    return plans


def configure(picam2, plan: CapturePlan, fps: float):
    sensor = {"output_size": plan.sensor_size} if plan.sensor_size else {}
    if plan.kind == "still":
        return picam2.create_still_configuration(main={"size": plan.main_size, "format": "BGR888"},
                                                 sensor=sensor)
    if plan.kind == "video":
        return picam2.create_video_configuration(main={"size": plan.main_size, "format": "BGR888"},
                                                 sensor=sensor, controls={"FrameRate": fps})
    return picam2.create_video_configuration(main={"size": plan.main_size, "format": "YUV420"},
                                             lores={"size": plan.lores_size, "format": "YUV420"},
                                             sensor=sensor, controls={"FrameRate": fps})


def capture(picam2, plan: CapturePlan, args: argparse.Namespace, stamp: str) -> dict:
    from PIL import Image  # noqa: PLC0415

    config = configure(picam2, plan, args.fps)
    picam2.configure(config)
    picam2.start()
    try:
        time.sleep(args.settle)  # let auto-exposure and white balance converge
        request = picam2.capture_request()
        try:
            metadata = request.get_metadata()
            if plan.kind == "motion":
                stream = picam2.stream_configuration("lores")
                width, height = stream["size"]
                stride = stream["stride"]
                buffer = request.make_buffer("lores")
                luma = buffer[: stride * height].reshape(height, stride)[:, :width]
                image = Image.fromarray(luma.copy())
                brightness = float(luma.mean())
            else:
                image = request.make_image("main")
                brightness = None
        finally:
            request.release()
    finally:
        picam2.stop()

    extension, save_kwargs = (("png", {"format": "PNG"}) if plan.kind == "motion"
                              else ("jpg", {"format": "JPEG", "quality": args.quality}))
    path = args.output / f"{stamp}_{plan.name}.{extension}"
    size_bytes = atomic_write(path, lambda handle: image.save(handle, **save_kwargs))

    with Image.open(path) as check:
        check.verify()

    return {
        "name": plan.name,
        "description": plan.description,
        "file": path.name,
        "size_bytes": size_bytes,
        "resolution": list(image.size),
        "sensor_output_size": list(plan.sensor_size) if plan.sensor_size else None,
        "exposure_us": metadata.get("ExposureTime"),
        "analogue_gain": metadata.get("AnalogueGain"),
        "lux": metadata.get("Lux"),
        "colour_temperature_k": metadata.get("ColourTemperature"),
        "mean_brightness": round(brightness, 1) if brightness is not None else None,
    }


def describe(result: dict) -> str:
    parts = [f"{result['resolution'][0]}x{result['resolution'][1]}",
             f"{result['size_bytes'] / 1024:.0f} KiB",
             f"exposure {result['exposure_us']} us",
             f"gain {result['analogue_gain']:.2f}" if isinstance(result["analogue_gain"], float)
             else f"gain {result['analogue_gain']}"]
    if result["mean_brightness"] is not None:
        parts.append(f"brightness {result['mean_brightness']:.0f}/255")
    return ", ".join(parts)


def build_arg_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(description="Phase 2: capture still images.")
    parser.add_argument("--camera", type=int, default=0, help="camera number (default 0)")
    parser.add_argument("--output", type=Path, default=DEFAULT_OUTPUT,
                        help=f"output directory (default {DEFAULT_OUTPUT})")
    parser.add_argument("--lores", type=parse_size, default=(640, 480),
                        help="motion detector stream size (default 640x480)")
    parser.add_argument("--fps", type=float, default=15.0, help="frame rate for video modes (default 15)")
    parser.add_argument("--settle", type=float, default=2.0,
                        help="seconds to let exposure settle before each capture (default 2)")
    parser.add_argument("--quality", type=int, default=90, help="JPEG quality 50-95 (default 90)")
    return parser


def main() -> int:
    args = build_arg_parser().parse_args()
    if os.geteuid() == 0:
        print("Do not run this as root. Type 'exit' to leave the root shell and run it as your user.",
              file=sys.stderr)
        return 2
    if not 1 <= args.fps <= 60 or not 0.5 <= args.settle <= 10 or not 50 <= args.quality <= 95:
        print("--fps must be 1-60, --settle 0.5-10, --quality 50-95", file=sys.stderr)
        return 2

    try:
        from picamera2 import Picamera2  # noqa: PLC0415
        import PIL  # noqa: F401, PLC0415
    except ImportError as exc:
        print(f"Missing dependency: {exc}\n"
              "Install with: sudo apt install -y python3-picamera2 python3-pil --no-install-recommends",
              file=sys.stderr)
        return 1

    args.output = args.output.expanduser().resolve()
    args.output.mkdir(parents=True, exist_ok=True)
    stamp = datetime.now(timezone.utc).strftime("%Y%m%dT%H%M%SZ")

    try:
        picam2 = Picamera2(args.camera)
    except (RuntimeError, IndexError) as exc:
        print(f"Could not open camera {args.camera}: {exc}\n"
              "Is another program using it? Check: pgrep -af 'rpicam|libcamera|picamera2'",
              file=sys.stderr)
        return 1

    results = []
    failures = 0
    try:
        model = picam2.camera_properties.get("Model")
        plans = build_plans(picam2.sensor_modes, args.lores)
        print(f"Camera: {model}. Saving to {args.output}\n")
        for plan in plans:
            print(f"Capturing {plan.name}: {plan.description} ...", flush=True)
            try:
                result = capture(picam2, plan, args, stamp)
            except Exception as exc:  # noqa: BLE001 - report and continue with the next capture
                failures += 1
                print(f"  [FAIL] {type(exc).__name__}: {exc}", flush=True)
                continue
            results.append(result)
            print(f"  [PASS] {result['file']}: {describe(result)}", flush=True)
    finally:
        picam2.close()

    summary = {
        "captured_at_utc": datetime.now(timezone.utc).isoformat(timespec="seconds"),
        "camera_model": model,
        "captures": results,
    }
    summary_path = args.output / f"{stamp}_captures.json"
    atomic_write(summary_path, lambda handle: handle.write(json.dumps(summary, indent=2).encode()))
    print(f"\nMetadata: {summary_path}")

    if failures:
        print(f"RESULT: {failures} capture(s) FAILED")
        return 1
    print("RESULT: STILL CAPTURE OK - Phase 2 complete.")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

## Test procedure

1. Point the camera at something with detail, ideally in **normal daylight or room light**, then run:

```bash
whoami                     # must print: ysak
python3 ~/surveillance/tools/phase2_still_capture.py
ls -la ~/surveillance/snapshots/phase2/
```

2. Copy the pictures to your computer and open them. Run this **on your laptop**, not the Pi (it works in the macOS, Linux and Windows PowerShell terminals):

```bash
scp 'ysak@ysak.local:~/surveillance/snapshots/phase2/*' .
```

3. Check the pictures:
   - **Focus.** The v1 lens is fixed-focus; some boards let you turn the lens.
   - **Orientation.** Is the image upside down?
   - **Colours.** Pinkish or purple in daylight means a NoIR camera.
   - **Wide vs crop1080.** Which view covers what you need?

## Expected output

```text
Camera: ov5647. Saving to /home/ysak/surveillance/snapshots/phase2

Capturing full: full sensor still 2592x1944 ...
  [PASS] 20260923T113000Z_full.jpg: 2592x1944, 900 KiB, exposure 20000 us, gain 2.00
Capturing wide: full field of view 1296x972 (binned) ...
  [PASS] 20260923T113000Z_wide.jpg: 1296x972, 300 KiB, ...
Capturing crop1080: 1920x1080 sensor mode ...
  [PASS] 20260923T113000Z_crop1080.jpg: 1920x1080, 450 KiB, ...
Capturing motion_view: motion detector view 640x480 ...
  [PASS] 20260923T113000Z_motion_view.png: 640x480, 150 KiB, ..., brightness 110/255

Metadata: /home/ysak/surveillance/snapshots/phase2/20260923T113000Z_captures.json
RESULT: STILL CAPTURE OK - Phase 2 complete.
```

File sizes and exposure values depend on the scene. The whole run takes about 10–15 seconds, and the filenames use UTC time (the `Z`).

## Troubleshooting

| Symptom | Fix |
|---|---|
| `Do not run this as root` | Type `exit` to leave the root shell, then run it again as `ysak`. |
| `Permission denied` on the snapshots folder | `sudo chown -R ysak:ysak /home/ysak/surveillance` |
| `Could not open camera` | Another program is using it: `pgrep -af 'rpicam\|picamera2'` |
| Images are very dark | Normal at night with this sensor. Retest in daylight. |
| Image is upside down | Tell me. We'll set a rotation option in the camera configuration in Phase 3. |

**Please send me:**
- the script's output;
- whether `wide` or `crop1080` suits your camera position better;
- whether the colours look normal or pinkish, and whether the board has IR LEDs;
- whether the image is upside down.

With those answers I'll lock in the recording resolution and tuning, then start Phase 3 (hardware H.264 recording).
