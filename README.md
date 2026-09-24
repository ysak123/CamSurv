I've fixed the tuning bug as version 0.11.2, and you can now switch between all the colour tuning options without restarting anything. In my test copy, the recorder and web interface went twice through every option, saving each one. Each time the camera reopened, libcamera used the file I selected, the dropdown kept all three choices, and a live snapshot was served. The fake camera I used reproduces your exact failure on the old code.

**What was wrong.** libcamera reads the tuning file only once, when its camera manager starts. Picamera2 passes the file name through an environment variable, `LIBCAMERA_RPI_TUNING_FILE`.

- **The tuning file never actually took effect.** My code asked "which cameras are connected?" before opening the camera. That started the camera manager before Picamera2 had set the variable. Any colours you compared in the last step were all using the default tuning.
- **The camera then failed.** Picamera2 had written your choice to a temporary file and pointed the variable at it. When the camera closed, that file was deleted but the variable still pointed to it. On the next open, libcamera couldn't find the file, rejected the camera, and reported 0 cameras.
- **Only a restart helped.** Picamera2 kept the broken "0 cameras" manager for the rest of the process, so every retry failed until the recorder was restarted.
- **The dropdown shrank.** It only lists files for the camera model the recorder reports. With the camera failing, there was no model, so only "Default" was left.

**What changed:**

- **`app/camera.py`:** before touching the camera, it finds the full path of the chosen file (or clears the setting for Default) and sets the variable. It also discards an idle camera manager, so a new one starts with the new tuning. If the file is missing or broken, you get a clear error.
- **`app/status.py` and `app/web_settings.py`:** the status now includes the last camera model that opened successfully. The dropdown uses it, falling back to the name of the current tuning file, so it keeps its options even while the camera is failing.

## Step 1 — Stop both programs

Press Ctrl+C in the recorder terminal and in the web terminal.

## Step 2 — Apply the update

```bash
cd ~/surveillance
cat > update_to_0_11_2.py <<'EOF'
#!/usr/bin/env python3
"""Update the surveillance project from 0.11.1 to 0.11.2 (run from ~/surveillance)."""
import shutil, sys
from pathlib import Path

EDITS = [
    ('app/__init__.py',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.11.1"\n',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.11.2"\n'),
    ('app/camera.py',
     'from __future__ import annotations\n\nimport logging\n\nfrom app.config import CameraSettings\n',
     'from __future__ import annotations\n\nimport gc\nimport json\nimport logging\nimport os\nfrom pathlib import Path\n\nfrom app.config import CameraSettings\n'),
    ('app/camera.py',
     'log = logging.getLogger("Camera")\n\n\nclass CameraError(RuntimeError):\n    pass\n\n\n',
     'log = logging.getLogger("Camera")\n\nTUNING_ENV = "LIBCAMERA_RPI_TUNING_FILE"\n# Same search order as Picamera2.load_tuning_file on a Raspberry Pi 4 (VC4 platform).\nTUNING_DIRS = (Path("~/libcamera/src/ipa/rpi/vc4/data").expanduser(),\n               Path("/usr/local/share/libcamera/ipa/rpi/vc4"),\n               Path("/usr/share/libcamera/ipa/rpi/vc4"))\n\n_last_model: str | None = None\n\n\nclass CameraError(RuntimeError):\n    pass\n\n\ndef find_tuning_file(name: str) -> Path | None:\n    for directory in TUNING_DIRS:\n        path = directory / name\n        if path.is_file():\n            return path\n    return None\n\n\ndef last_opened_model() -> str | None:\n    """Sensor model of the last camera this process opened successfully (kept while the camera is failing)."""\n    return _last_model\n\n\ndef _select_tuning(picamera2_class, tuning_path: Path | None) -> None:\n    """libcamera reads the tuning file only when its camera manager is created, and Picamera2 keeps one\n    manager until its last camera closes (or forever, if it found no cameras). So the variable must be set\n    before anything touches the manager, and an idle manager must be discarded for a change to apply."""\n    if tuning_path is None:\n        os.environ.pop(TUNING_ENV, None)\n    else:\n        os.environ[TUNING_ENV] = str(tuning_path)\n    manager = getattr(picamera2_class, "_cm", None)\n    if manager is not None and hasattr(manager, "_cms") and not getattr(manager, "cameras", None):\n        manager._cms = None\n        gc.collect()\n\n\n'),
    ('app/camera.py',
     '\n        s = self.settings\n        cameras = Picamera2.global_camera_info()\n        if s.camera_num >= len(cameras):\n            raise CameraError(f"camera {s.camera_num} not detected ({len(cameras)} camera(s) found)")\n        tuning = None\n        if s.tuning_file:\n            try:\n                tuning = Picamera2.load_tuning_file(s.tuning_file)\n            except (RuntimeError, OSError, ValueError) as exc:\n                raise CameraError(f"cannot load tuning file {s.tuning_file}: {exc}") from exc\n        try:\n            self.picam2 = Picamera2(s.camera_num, tuning=tuning)\n        except (RuntimeError, IndexError) as exc:\n            reason = str(exc).rstrip(".")\n',
     '\n        s = self.settings\n        tuning_path = None\n        if s.tuning_file:\n            tuning_path = find_tuning_file(s.tuning_file)\n            if tuning_path is None:\n                raise CameraError(f"tuning file {s.tuning_file} not found in {\', \'.join(map(str, TUNING_DIRS))}")\n            try:\n                json.loads(tuning_path.read_text())\n            except (OSError, ValueError) as exc:\n                raise CameraError(f"cannot read tuning file {tuning_path}: {exc}") from exc\n        _select_tuning(Picamera2, tuning_path)\n\n        cameras = Picamera2.global_camera_info()\n        if s.camera_num >= len(cameras):\n            hint = f"; the tuning file {s.tuning_file} may not suit this camera" if s.tuning_file else ""\n            raise CameraError(f"camera {s.camera_num} not detected ({len(cameras)} camera(s) found{hint})")\n        try:\n            self.picam2 = Picamera2(s.camera_num, tuning=str(tuning_path) if tuning_path else None)\n        except (RuntimeError, IndexError) as exc:\n            reason = str(exc).rstrip(".")\n'),
    ('app/camera.py',
     '            raise CameraError(f"cannot configure camera: {exc}") from exc\n\n        self.model = self.picam2.camera_properties.get("Model", "unknown")\n        log.info("Opened %s: main %dx%d, lores %dx%d, %g fps, sensor mode %s%s", self.model, s.width, s.height,\n                 s.lores_width, s.lores_height, s.framerate,\n',
     '            raise CameraError(f"cannot configure camera: {exc}") from exc\n\n        global _last_model\n        self.model = _last_model = self.picam2.camera_properties.get("Model", "unknown")\n        log.info("Opened %s: main %dx%d, lores %dx%d, %g fps, sensor mode %s%s", self.model, s.width, s.height,\n                 s.lores_width, s.lores_height, s.framerate,\n'),
    ('app/status.py',
     '\nfrom app import __version__\nfrom app.fileutil import atomic_write_bytes\n\n',
     '\nfrom app import __version__\nfrom app.camera import last_opened_model\nfrom app.fileutil import atomic_write_bytes\n\n'),
    ('app/status.py',
     '            "camera": {"name": cam.name, "resolution": f"{cam.width}x{cam.height}", "fps": cam.framerate,\n                       "bitrate": cam.bitrate, "model": recorder.camera_model if recorder else None,\n                       "tuning_file": cam.tuning_file},\n            "recording": {\n                "active": recording and not recorder.paused,\n',
     '            "camera": {"name": cam.name, "resolution": f"{cam.width}x{cam.height}", "fps": cam.framerate,\n                       "bitrate": cam.bitrate, "model": recorder.camera_model if recorder else None,\n                       "last_model": last_opened_model(), "tuning_file": cam.tuning_file},\n            "recording": {\n                "active": recording and not recorder.paused,\n'),
    ('app/web_settings.py',
     '\nfrom app.auth import AuthError, AuthStore\nfrom app.config import (ALLOWED_SEGMENT_SECONDS, LIVE_QUALITIES, LOG_LEVELS, PROJECT_ROOT, RETENTION_MODES,\n                        ConfigError, Settings, diff_settings, load_settings, save_settings, settings_from_dict)\n',
     '\nfrom app.auth import AuthError, AuthStore\nfrom app.camera import TUNING_DIRS\nfrom app.config import (ALLOWED_SEGMENT_SECONDS, LIVE_QUALITIES, LOG_LEVELS, PROJECT_ROOT, RETENTION_MODES,\n                        ConfigError, Settings, diff_settings, load_settings, save_settings, settings_from_dict)\n'),
    ('app/web_settings.py',
     'REAUTH_MAX_AGE_S = 300\nRESTART_DELAY_S = 1.5\nTUNING_DIRS = (Path("/usr/share/libcamera/ipa/rpi/vc4"), Path("/usr/local/share/libcamera/ipa/rpi/vc4"))\nRESOLUTIONS = {\n    "1296x972": (1296, 972, 640, 480, "1296 × 972 – full field of view (recommended for OV5647)"),\n',
     'REAUTH_MAX_AGE_S = 300\nRESTART_DELAY_S = 1.5\nRESOLUTIONS = {\n    "1296x972": (1296, 972, 640, 480, "1296 × 972 – full field of view (recommended for OV5647)"),\n'),
    ('app/web_settings.py',
     '\n\ndef tuning_choices(model: str | None, current: str) -> list[tuple[str, str]]:\n    names = set()\n    for directory in TUNING_DIRS:\n        if directory.is_dir():\n            names.update(p.name for p in directory.glob("*.json") if model and p.stem.startswith(model))\n    if current:\n        names.add(current)\n',
     '\n\ndef camera_model(status: dict | None, current_tuning: str) -> str | None:\n    """Sensor model for filtering tuning files, also while the recorder cannot open the camera."""\n    camera = (status or {}).get("camera") or {}\n    model = camera.get("model") or camera.get("last_model")\n    if not model and current_tuning:\n        model = current_tuning.removesuffix(".json").split("_")[0]\n    return model or None\n\n\ndef tuning_choices(model: str | None, current: str) -> list[tuple[str, str]]:\n    """Tuning files for this sensor; every file if the sensor is unknown (camera never opened)."""\n    names = set()\n    for directory in TUNING_DIRS:\n        if directory.is_dir():\n            names.update(p.name for p in directory.glob("*.json")\n                         if not model or p.stem == model or p.stem.startswith(model + "_"))\n    if current:\n        names.add(current)\n'),
    ('app/web_settings.py',
     '\n    def render(data: dict, current: Settings, *, errors=(), message=None, changes=(), reloading=False, status=200):\n        recorder = read_status(settings.runtime_dir) or {}\n        model = (recorder.get("camera") or {}).get("model")\n        tuning = tuning_choices(model, data["camera"]["tuning_file"])\n        return render_template(\n',
     '\n    def render(data: dict, current: Settings, *, errors=(), message=None, changes=(), reloading=False, status=200):\n        model = camera_model(read_status(settings.runtime_dir), current.camera.tuning_file)\n        tuning = tuning_choices(model, data["camera"]["tuning_file"])\n        return render_template(\n'),
    ('app/web_settings.py',
     '            return render(asdict(current), current, errors=[problem], status=409)\n        data = asdict(current)\n        model = ((read_status(settings.runtime_dir) or {}).get("camera") or {}).get("model")\n        errors = apply_form(data, request.form, tuning_choices(model, current.camera.tuning_file))\n        if errors:\n',
     '            return render(asdict(current), current, errors=[problem], status=409)\n        data = asdict(current)\n        model = camera_model(read_status(settings.runtime_dir), current.camera.tuning_file)\n        errors = apply_form(data, request.form, tuning_choices(model, current.camera.tuning_file))\n        if errors:\n'),
]

texts = {}
for rel, old, new in EDITS:
    text = texts.setdefault(rel, Path(rel).read_text())
    if text.count(old) != 1:
        sys.exit(f"ABORTED, nothing changed: {rel} does not match the expected 0.11.1 code "
                 f"(found {text.count(old)} matches for:\n{old})")
    texts[rel] = text.replace(old, new)
for rel, text in texts.items():
    shutil.copy2(rel, rel + ".bak")
    Path(rel).write_text(text)
    print(f"updated {rel}  (backup: {rel}.bak)")
print("Update to 0.11.2 complete.")
EOF
python3 update_to_0_11_2.py
sha256sum app/camera.py app/status.py app/web_settings.py app/__init__.py
```

Expected output:

```
updated app/__init__.py  (backup: app/__init__.py.bak)
updated app/camera.py  (backup: app/camera.py.bak)
updated app/status.py  (backup: app/status.py.bak)
updated app/web_settings.py  (backup: app/web_settings.py.bak)
Update to 0.11.2 complete.
cf187ac89b078c352d92ea52bb260f5227fe4388c462b2e9a08e4bd859f6cdd0  app/camera.py
1de7d0034dc1a3cf9f02315627b5048cceed33e502a437f263d8566eddf94a47  app/status.py
0f9ef7d54bb34df4e8cdaf9a516fbaac2843218ff467d44c4d8ef293ced86629  app/web_settings.py
a9d443bebc6f40fd5099d39d7d818013ec132abfca0aa3f2ae9b64de338b2f47  app/__init__.py
```

If a checksum differs, part of the script didn't paste cleanly. Tell me which file, and restore from its `.bak`. If you see "ABORTED", nothing was changed.

## Step 3 — Start both programs again

Start the recorder and the web interface the same way as before. Keep the SSH tunnel open and go to http://localhost:8080.

## Step 4 — Retest the dropdown

On the Settings page, pick a colour tuning option and save it. Then wait about 10 seconds and check Live View. Go through all options twice:

Default → `ov5647.json` → `ov5647_noir.json` → Default → `ov5647_noir.json` → `ov5647.json`

Each time:
- **Dashboard:** recording, with no error.
- **Live View:** shows a picture.
- **Dropdown:** still lists all three options.

Then check the log:

```bash
grep -E "Opened ov5647|not detected|tuning" ~/surveillance/logs/surveillance.log | tail -12
```

Expected: one line like this for each save, ending with the file you picked (no ending for Default), and no "not detected" lines:

```
INFO Camera: Opened ov5647: main 1296x972, lores 640x480, 15 fps, sensor mode 1296x972, tuning ov5647_noir.json
```

## Step 5 — Compare the colours again

This is the first time the tuning files actually take effect. So in daylight, compare Default, `ov5647.json` and `ov5647_noir.json` in Live View, and keep the one that looks most natural. With a NoIR camera, `ov5647_noir.json` should reduce the pink tint. It can't remove it completely, because the lens has no infrared filter.

Send me the Step 2 output and the Step 4 log lines. Tell me which tuning you chose. Once this works, we'll move on to Phase 12, secure remote access with Tailscale.
