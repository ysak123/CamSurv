Phase 10, the live view, is ready for you to test on the Pi. It passed all my tests here with the fake camera, but it hasn't touched your real camera yet. You'll need to restart both the recorder and the web interface after updating.

## What Phase 10 adds

**Live View page**
- Video from the 640×480 low-resolution stream at about **7.5 frames/s**, compressed by the Pi 4's **hardware JPEG encoder**. The delay should be well under a second.
- Standard MJPEG shown in a plain image element, so it works in every browser and uses your existing local-only connection.
- A red **● LIVE** badge while frames arrive, plus **Pause** and **Full screen** buttons.
- **The video only runs while someone is watching:**
  - The camera starts encoding when the first viewer connects and stops when the last one leaves.
  - The page closes the stream when you switch tabs or minimise the window, and after 10 minutes. You then get a **Resume** button.
- **At most 3 viewers at once.** A 4th sees "Camera not reachable (… or too many viewers). Retrying…".

**Dashboard**
- The placeholder box now shows the **latest camera image, refreshed every 10 s**. Tap it to open the live view.
- Each snapshot is a single frame compressed in software, so the live encoder isn't started and stopped every 10 s.

**Safety for recording**
- The live view encodes the low-resolution stream the motion detector already uses, so the recording itself isn't touched.
- If anything in the live view fails, it's logged and recording carries on. If the hardware JPEG encoder can't start, it falls back to software JPEG automatically.
- The web app talks to the recorder through a socket file readable only by your user (`srw-------`) in the RAM folder.

**New settings** (in the `live` section):

| Setting | Default | Range |
|---|---|---|
| `enabled` | true | true / false |
| `max_fps` | 8 | 1–30 |
| `quality` | medium | low / medium / high |
| `max_viewers` | 3 | 1–6 |
| `max_view_minutes` | 10 | 1–240 |

## What I tested here

This machine has no hardware JPEG encoder, which conveniently exercised the software fallback.

- **Snapshot:** a valid 640×480 JPEG. **Stream:** 29 frames in 4 s, which is about 7.2 frames/s against the 7.5 target.
- **Viewer limit:** 3 viewers were accepted and the 4th got `503`. After 20 connections that were cut off before their first frame, a new viewer still got in. **A leak I found and fixed:** a viewer that disconnected before the first frame would have used up a slot permanently.
- **In Chrome:**
  - The LIVE badge appeared.
  - **Pause** made the recorder log `Live view stopped (no viewers)`, and **Resume** restarted it.
  - Leaving the page for the dashboard stopped it too.
  - No console errors.
- **Recorder stopped:** snapshot and stream both return `503`, and the socket file is removed.
- **Snapshot colours:** the dashboard snapshot came out **green** in my test. That was my fake camera, whose buffers have empty colour data. I checked the colour conversion on its own with a constructed image: grey came out as (120, 120, 120) and a red area as red.
- **Update script:** turns your 0.9.0 files into files identical to the tested ones, and aborts safely if run twice.

## Update the files

No new packages are needed: `simplejpeg` comes with Picamera2. You can check with `python3 -c "import simplejpeg; print('ok')"`.

**1. Stop the recorder and the web interface** (Ctrl+C in both windows). Then apply the update script, which changes 8 files and makes `.bak` backups:

```bash
cd ~/surveillance
cat > update_to_0_10_0.py <<'EOF'
#!/usr/bin/env python3
"""Update the surveillance project from 0.9.0 to 0.10.0 (run from ~/surveillance)."""
import shutil, sys
from pathlib import Path

EDITS = [
    ('app/__init__.py',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.9.0"\n',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.10.0"\n'),
    ('app/config.py',
     'RETENTION_MODES = ("oldest_first",)\nLOCAL_WEB_HOSTS = ("127.0.0.1", "::1", "localhost")\nMAX_ENCODER_PIXELS = 1920 * 1080\nCAMERA_NAME_PATTERN = re.compile(r"^[\\w][\\w .,\'()-]{0,39}$")\n',
     'RETENTION_MODES = ("oldest_first",)\nLOCAL_WEB_HOSTS = ("127.0.0.1", "::1", "localhost")\nLIVE_QUALITIES = ("low", "medium", "high")\nMAX_ENCODER_PIXELS = 1920 * 1080\nCAMERA_NAME_PATTERN = re.compile(r"^[\\w][\\w .,\'()-]{0,39}$")\n'),
    ('app/config.py',
     '\n@dataclass(frozen=True)\nclass WebSettings:\n    host: str = "127.0.0.1"\n',
     '\n@dataclass(frozen=True)\nclass LiveSettings:\n    enabled: bool = True\n    max_fps: float = 8.0\n    quality: str = "medium"\n    max_viewers: int = 3\n    max_view_minutes: int = 10\n\n    def validate(self) -> None:\n        _check_range("live.max_fps", self.max_fps, 1.0, 30.0)\n        if self.quality not in LIVE_QUALITIES:\n            raise ConfigError(f"live.quality must be one of: {\', \'.join(LIVE_QUALITIES)}")\n        _check_range("live.max_viewers", self.max_viewers, 1, 6)\n        _check_range("live.max_view_minutes", self.max_view_minutes, 1, 240)\n\n\n@dataclass(frozen=True)\nclass WebSettings:\n    host: str = "127.0.0.1"\n'),
    ('app/config.py',
     '    storage: StorageSettings = field(default_factory=StorageSettings)\n    motion: MotionSettings = field(default_factory=MotionSettings)\n    web: WebSettings = field(default_factory=WebSettings)\n    paths: PathSettings = field(default_factory=PathSettings)\n',
     '    storage: StorageSettings = field(default_factory=StorageSettings)\n    motion: MotionSettings = field(default_factory=MotionSettings)\n    live: LiveSettings = field(default_factory=LiveSettings)\n    web: WebSettings = field(default_factory=WebSettings)\n    paths: PathSettings = field(default_factory=PathSettings)\n'),
    ('app/recorder.py',
     'from app.config import Settings\nfrom app.fileutil import fsync_directory, fsync_file\n\nlog = logging.getLogger("Recorder")\n',
     'from app.config import Settings\nfrom app.fileutil import fsync_directory, fsync_file\nfrom app.streaming import LiveServer\n\nlog = logging.getLogger("Recorder")\n'),
    ('app/recorder.py',
     '        self._motion = None\n        self._motion_listeners: list[Callable] = []\n        self.last_segment: Segment | None = None\n\n',
     '        self._motion = None\n        self._motion_listeners: list[Callable] = []\n        self._live = None\n        self.last_segment: Segment | None = None\n\n'),
    ('app/recorder.py',
     '                                               self._motion_listeners)\n            self._motion.start()\n\n    @staticmethod\n',
     '                                               self._motion_listeners)\n            self._motion.start()\n        if self.settings.live.enabled:\n            self._start_live()\n\n    def _start_live(self) -> None:\n        try:\n            self._live = LiveServer(self._camera.picam2, self.settings)\n            self._live.start()\n        except Exception as exc:  # noqa: BLE001 - live view is optional; recording must continue\n            log.error("Live view unavailable: %s", exc)\n            self._live = None\n\n    @staticmethod\n'),
    ('app/recorder.py',
     '    def stop(self) -> None:\n        was_recording = self._recording\n        if self._motion is not None:\n            self._motion.stop()\n',
     '    def stop(self) -> None:\n        was_recording = self._recording\n        if self._live is not None:\n            self._live.close()\n            self._live = None\n        if self._motion is not None:\n            self._motion.stop()\n'),
    ('app/web.py',
     'from app.status import read_status\nfrom app.system_info import SystemInfo\nfrom app.web_recordings import create_blueprint\n\n',
     'from app.status import read_status\nfrom app.system_info import SystemInfo\nfrom app.web_live import create_blueprint as create_live_blueprint\nfrom app.web_recordings import create_blueprint\n\n'),
    ('app/web.py',
     'NAVIGATION = [\n    ("dashboard", "Dashboard"),\n    ("live", "Live View"),\n    ("rec.recordings", "Recordings"),\n    ("rec.events", "Motion Events"),\n',
     'NAVIGATION = [\n    ("dashboard", "Dashboard"),\n    ("live.live_page", "Live View"),\n    ("rec.recordings", "Recordings"),\n    ("rec.events", "Motion Events"),\n'),
    ('app/web.py',
     '    app.config.update(JSON_SORT_KEYS=False, MAX_CONTENT_LENGTH=64 * 1024)\n    app.register_blueprint(create_blueprint(settings))\n    system_info = SystemInfo()\n\n',
     '    app.config.update(JSON_SORT_KEYS=False, MAX_CONTENT_LENGTH=64 * 1024)\n    app.register_blueprint(create_blueprint(settings))\n    app.register_blueprint(create_live_blueprint(settings))\n    system_info = SystemInfo()\n\n'),
    ('app/web.py',
     '        return render_template("settings.html", title="Settings", page="settings_page", sections=asdict(settings))\n\n    @app.get("/live")\n    def live():\n        return render_template("placeholder.html", title="Live View", page="live", phase=10,\n                               text="Live view is added in Phase 10.")\n\n    @app.get("/api/status")\n    def api_status():\n',
     '        return render_template("settings.html", title="Settings", page="settings_page", sections=asdict(settings))\n\n    @app.get("/api/status")\n    def api_status():\n'),
    ('app/web_main.py',
     '\nWEB_LOG_FILE = "web.log"\nWORKER_THREADS = 6\n\n\n',
     '\nWEB_LOG_FILE = "web.log"\n# Each live viewer holds one worker thread for as long as it watches; keep spare threads for pages.\nSPARE_WORKER_THREADS = 4\n\n\n'),
    ('app/web_main.py',
     '    host, port = settings.web.host, settings.web.port\n    log.info("Web interface %s listening on http://%s:%d (local only)", __version__, host, port)\n    serve(create_app(settings), host=host, port=port, threads=WORKER_THREADS, ident="",\n          clear_untrusted_proxy_headers=True)\n    return 0\n\n',
     '    host, port = settings.web.host, settings.web.port\n    log.info("Web interface %s listening on http://%s:%d (local only)", __version__, host, port)\n    serve(create_app(settings), host=host, port=port, threads=settings.live.max_viewers + SPARE_WORKER_THREADS,\n          ident="", clear_untrusted_proxy_headers=True)\n    return 0\n\n'),
    ('web/templates/dashboard.html',
     '{% block content %}\n<section class="live-box" aria-label="Live camera">\n  <div class="live-placeholder">\n    <span class="live-title">LIVE CAMERA</span>\n    <span class="muted">Live view is added in Phase 10</span>\n  </div>\n</section>\n\n',
     '{% block content %}\n<section class="live-box" aria-label="Live camera">\n  <a class="snapshot-link" href="{{ url_for(\'live.live_page\') }}">\n    <img id="snapshot" class="snapshot" alt="Latest camera image" hidden>\n    <span class="live-placeholder" id="snapshot-placeholder">\n      <span class="live-title">LIVE CAMERA</span>\n      <span class="muted" id="snapshot-message">Loading the latest image…</span>\n    </span>\n    <span class="snapshot-hint">Open live view &rarr;</span>\n  </a>\n</section>\n\n'),
    ('web/static/app.js',
     '  });\n  refresh();\n})();\n',
     '  });\n  refresh();\n\n  // Dashboard: a still image refreshed every 10 s (a single frame, not the live encoder).\n  const snapshot = document.getElementById("snapshot");\n  if (snapshot) {\n    const placeholder = document.getElementById("snapshot-placeholder");\n    const note = document.getElementById("snapshot-message");\n    snapshot.addEventListener("load", () => { snapshot.hidden = false; placeholder.hidden = true; });\n    snapshot.addEventListener("error", () => {\n      snapshot.hidden = true;\n      placeholder.hidden = false;\n      note.textContent = "Camera image not available";\n    });\n    const refreshSnapshot = () => {\n      if (document.visibilityState === "visible") snapshot.src = `/live/snapshot.jpg?ts=${Date.now()}`;\n    };\n    refreshSnapshot();\n    setInterval(refreshSnapshot, 10000);\n  }\n})();\n'),
    ('web/static/style.css',
     '  table.list td.actions { flex-direction: column; align-items: stretch; }\n}\n',
     '  table.list td.actions { flex-direction: column; align-items: stretch; }\n}\n\n/* ---- Phase 10: live view ---- */\n[hidden] { display: none !important; }\n\n.snapshot-link {\n  position: relative;\n  display: grid;\n  place-items: center;\n  width: 100%;\n  height: 100%;\n  color: inherit;\n  text-decoration: none;\n}\n.snapshot { width: 100%; height: 100%; object-fit: contain; border-radius: var(--radius); background: #000; }\n.snapshot-hint {\n  position: absolute;\n  right: .75rem;\n  bottom: .75rem;\n  padding: .25rem .6rem;\n  border-radius: 6px;\n  background: rgba(0, 0, 0, .65);\n  font-size: .8rem;\n}\n\n.live-frame {\n  position: relative;\n  width: 100%;\n  aspect-ratio: 4 / 3;\n  max-height: 75vh;\n  margin: 0 auto;\n  overflow: hidden;\n  border-radius: 8px;\n  background: #000;\n  display: grid;\n  place-items: center;\n}\n.live-frame img { width: 100%; height: 100%; object-fit: contain; }\n.live-frame:fullscreen { max-height: none; border-radius: 0; }\n.live-overlay {\n  position: absolute;\n  inset: 0;\n  display: grid;\n  place-content: center;\n  justify-items: center;\n  gap: .75rem;\n  padding: 1rem;\n  text-align: center;\n  background: rgba(0, 0, 0, .6);\n}\n.live-badge {\n  position: absolute;\n  left: .75rem;\n  top: .75rem;\n  padding: .2rem .55rem;\n  border-radius: 6px;\n  background: rgba(0, 0, 0, .65);\n  color: var(--bad);\n  font-size: .8rem;\n  font-weight: 700;\n  letter-spacing: .05em;\n}\n'),
]

texts = {}
for rel, old, new in EDITS:
    text = texts.setdefault(rel, Path(rel).read_text())
    if text.count(old) != 1:
        sys.exit(f"ABORTED, nothing changed: {rel} does not match the expected 0.9.0 code "
                 f"(found {text.count(old)} matches for:\n{old})")
    texts[rel] = text.replace(old, new)
for rel, text in texts.items():
    shutil.copy2(rel, rel + ".bak")
    Path(rel).write_text(text)
    print(f"updated {rel}  (backup: {rel}.bak)")
print("Update to 0.10.0 complete.")
EOF
python3 update_to_0_10_0.py
```

It should print eight `updated …` lines and then `Update to 0.10.0 complete.`

**2. New file `app/streaming.py`** (the recorder side):

```bash
cat > ~/surveillance/app/streaming.py <<'EOF'
"""Live view source inside the recorder process (Phase 10).

The recorder listens on a Unix socket (<runtime_dir>/live.sock, owner-only permissions).
The web process connects and sends one command line:

    STREAM\\n    -> a stream of JPEG frames until either side disconnects
    SNAPSHOT\\n  -> exactly one JPEG frame

Each frame is sent as a 4-byte big-endian length followed by the JPEG bytes.

The JPEG encoder only runs while at least one STREAM client is connected. It encodes the
low-resolution stream that the camera already produces for motion detection, using the
Pi 4's hardware JPEG encoder (software JPEG as fallback), so the recording is not touched.
Every client only ever receives the newest frame, so a slow viewer cannot hold anything up.
"""
from __future__ import annotations

import logging
import math
import os
import socket
import struct
import threading
from pathlib import Path

import simplejpeg
from picamera2.encoders import JpegEncoder, MJPEGEncoder, Quality
from picamera2.outputs import Output

log = logging.getLogger("LiveStream")

SOCKET_NAME = "live.sock"
COMMAND_MAX_BYTES = 32
CLIENT_TIMEOUT_S = 10.0
FRAME_WAIT_S = 5.0
SNAPSHOT_QUALITY = 85
SOFTWARE_JPEG_QUALITY = 70
QUALITY_LEVELS = {"low": Quality.LOW, "medium": Quality.MEDIUM, "high": Quality.HIGH}


def send_frame(conn: socket.socket, data: bytes) -> None:
    conn.sendall(struct.pack(">I", len(data)) + data)


def yuv420_to_jpeg(buffer, width: int, height: int, stride: int, quality: int) -> bytes:
    frame = buffer.reshape(height * 3 // 2, stride)
    y = frame[:height, :width]
    chroma = frame.reshape(frame.shape[0] * 2, stride // 2)
    u = chroma[2 * height: 2 * height + height // 2, : width // 2]
    v = chroma[2 * height + height // 2:, : width // 2]
    return simplejpeg.encode_jpeg_yuv_planes(y, u, v, quality)


class _FrameHub(Output):
    """Picamera2 output that keeps only the newest JPEG frame and wakes waiting clients."""

    def __init__(self) -> None:
        super().__init__()
        self._condition = threading.Condition()
        self._frame: bytes | None = None
        self._sequence = 0

    def outputframe(self, frame, keyframe=True, timestamp=None, packet=None, audio=False) -> None:
        with self._condition:
            self._frame = bytes(frame)
            self._sequence += 1
            self._condition.notify_all()

    def next_frame(self, after: int, timeout: float) -> tuple[int, bytes | None]:
        with self._condition:
            self._condition.wait_for(lambda: self._sequence != after, timeout=timeout)
            return self._sequence, self._frame if self._sequence != after else None


class LiveServer:
    def __init__(self, picam2, settings) -> None:
        self._picam2 = picam2
        self._live = settings.live
        self._framerate = settings.camera.framerate
        self._path: Path = settings.runtime_dir / SOCKET_NAME
        self._hub = _FrameHub()
        self._encoder = None
        self._clients = 0
        self._lock = threading.Lock()
        self._closed = threading.Event()
        self._server: socket.socket | None = None
        self._thread = threading.Thread(target=self._accept_loop, name="live-accept", daemon=True)

    # ------------------------------------------------------------ lifecycle
    def start(self) -> None:
        self._path.parent.mkdir(parents=True, exist_ok=True, mode=0o700)
        self._path.unlink(missing_ok=True)
        server = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
        old_umask = os.umask(0o177)  # the socket file is created owner-only (0600)
        try:
            server.bind(str(self._path))
        finally:
            os.umask(old_umask)
        server.listen(8)
        self._server = server
        self._thread.start()
        log.info("Live view ready (up to %g frames/s, %s quality, on demand)", self._max_fps(), self._live.quality)

    def close(self) -> None:
        self._closed.set()
        if self._server is not None:
            try:
                self._server.close()
            except OSError:
                pass
        self._path.unlink(missing_ok=True)
        with self._lock:
            self._stop_encoder()

    # ------------------------------------------------------------- encoder
    def _max_fps(self) -> float:
        return min(self._live.max_fps, self._framerate)

    def _start_encoder(self) -> bool:
        if self._encoder is not None:
            return True
        skip = max(1, math.ceil(self._framerate / self._max_fps()))
        try:
            encoder = MJPEGEncoder()
            encoder.frame_skip_count = skip
            self._picam2.start_encoder(encoder, self._hub, name="lores",
                                       quality=QUALITY_LEVELS[self._live.quality])
            kind = "hardware"
        except Exception as exc:  # noqa: BLE001 - fall back to software JPEG
            log.warning("Hardware JPEG encoder unavailable (%s); using software JPEG", exc)
            try:
                encoder = JpegEncoder(num_threads=2, q=SOFTWARE_JPEG_QUALITY)
                encoder.frame_skip_count = skip
                self._picam2.start_encoder(encoder, self._hub, name="lores")
                kind = "software"
            except Exception as exc2:  # noqa: BLE001
                log.error("Cannot start the live view encoder: %s", exc2)
                return False
        self._encoder = encoder
        log.info("Live view started (%s JPEG, %.1f frames/s)", kind, self._framerate / skip)
        return True

    def _stop_encoder(self) -> None:
        if self._encoder is None:
            return
        encoder, self._encoder = self._encoder, None
        try:
            self._picam2.stop_encoder(encoder)
        except Exception as exc:  # noqa: BLE001
            log.debug("Error stopping the live view encoder: %s", exc)
        log.info("Live view stopped (no viewers)")

    # ------------------------------------------------------------- clients
    def _accept_loop(self) -> None:
        while not self._closed.is_set():
            try:
                conn, _ = self._server.accept()
            except OSError:
                return
            threading.Thread(target=self._handle, args=(conn,), name="live-client", daemon=True).start()

    def _handle(self, conn: socket.socket) -> None:
        with conn:
            conn.settimeout(CLIENT_TIMEOUT_S)
            try:
                command = conn.recv(COMMAND_MAX_BYTES).strip()
                if command == b"SNAPSHOT":
                    send_frame(conn, self._snapshot())
                elif command == b"STREAM":
                    self._stream(conn)
            except (OSError, TimeoutError):
                pass
            except Exception:  # noqa: BLE001 - a viewer must never affect recording
                log.exception("Live view client failed")

    def _snapshot(self) -> bytes:
        stream = self._picam2.stream_configuration("lores")
        width, height = stream["size"]
        buffer = self._picam2.capture_buffer("lores", wait=FRAME_WAIT_S)
        return yuv420_to_jpeg(buffer, width, height, stream["stride"], SNAPSHOT_QUALITY)

    def _stream(self, conn: socket.socket) -> None:
        with self._lock:
            if self._clients >= self._live.max_viewers:
                return
            if not self._start_encoder():
                return
            self._clients += 1
        try:
            sequence = 0
            while not self._closed.is_set():
                sequence, frame = self._hub.next_frame(sequence, FRAME_WAIT_S)
                if frame is None:
                    continue
                send_frame(conn, frame)
        finally:
            with self._lock:
                self._clients -= 1
                if self._clients == 0:
                    self._stop_encoder()
EOF
```

**3. New file `app/web_live.py`** (the web side):

```bash
cat > ~/surveillance/app/web_live.py <<'EOF'
"""Live view in the browser (Phase 10).

Relays JPEG frames from the recorder's Unix socket to the browser as MJPEG
(multipart/x-mixed-replace), which every browser shows in a plain <img> element.
The recorder only encodes while at least one viewer is connected.
"""
from __future__ import annotations

import socket
import struct
import threading

from flask import Blueprint, Response, abort, render_template

from app.config import Settings

SOCKET_NAME = "live.sock"
CONNECT_TIMEOUT_S = 3.0
READ_TIMEOUT_S = 10.0
MAX_FRAME_BYTES = 4 * 1024 * 1024
BOUNDARY = "frame"


def _read_exact(conn: socket.socket, size: int) -> bytes:
    chunks, remaining = [], size
    while remaining:
        chunk = conn.recv(min(remaining, 65536))
        if not chunk:
            raise ConnectionError("recorder closed the live stream")
        chunks.append(chunk)
        remaining -= len(chunk)
    return b"".join(chunks)


def _read_frame(conn: socket.socket) -> bytes:
    (length,) = struct.unpack(">I", _read_exact(conn, 4))
    if not 0 < length <= MAX_FRAME_BYTES:
        raise ConnectionError(f"invalid frame length {length}")
    return _read_exact(conn, length)


class _MjpegStream:
    """Response body. close() is always called by the WSGI server, even if the client
    disconnects before the first frame, so the viewer slot and socket are never leaked."""

    def __init__(self, conn: socket.socket, release) -> None:
        self._conn = conn
        self._release = release
        self._closed = False

    def __iter__(self):
        try:
            while True:
                frame = _read_frame(self._conn)
                yield (f"--{BOUNDARY}\r\nContent-Type: image/jpeg\r\nContent-Length: {len(frame)}\r\n\r\n"
                       .encode() + frame + b"\r\n")
        except (OSError, ConnectionError, struct.error):
            return
        finally:
            self.close()

    def close(self) -> None:
        if self._closed:
            return
        self._closed = True
        self._conn.close()
        self._release()


def create_blueprint(settings: Settings) -> Blueprint:
    bp = Blueprint("live", __name__)
    socket_path = str(settings.runtime_dir / SOCKET_NAME)
    viewers = threading.BoundedSemaphore(settings.live.max_viewers)

    def connect(command: bytes) -> socket.socket:
        if not settings.live.enabled:
            abort(404)
        conn = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
        conn.settimeout(CONNECT_TIMEOUT_S)
        try:
            conn.connect(socket_path)
            conn.sendall(command + b"\n")
        except OSError:
            conn.close()
            abort(503)
        conn.settimeout(READ_TIMEOUT_S)
        return conn

    @bp.get("/live")
    def live_page():
        return render_template("live.html", title="Live View", page="live.live_page", live=settings.live)

    @bp.get("/live/snapshot.jpg")
    def snapshot():
        conn = connect(b"SNAPSHOT")
        try:
            frame = _read_frame(conn)
        except (OSError, ConnectionError, struct.error):
            abort(503)
        finally:
            conn.close()
        return Response(frame, mimetype="image/jpeg")

    @bp.get("/live/stream")
    def stream():
        if not viewers.acquire(blocking=False):
            abort(503)
        try:
            conn = connect(b"STREAM")
        except Exception:
            viewers.release()
            raise
        response = Response(_MjpegStream(conn, viewers.release),
                            mimetype=f"multipart/x-mixed-replace; boundary={BOUNDARY}")
        response.headers["X-Accel-Buffering"] = "no"
        return response

    return bp
EOF
```

**4. The Live View page and its script:**

```bash
cat > ~/surveillance/web/templates/live.html <<'EOF'
{% extends "base.html" %}
{% block scripts %}<script src="{{ url_for('static', filename='live.js') }}" defer></script>{% endblock %}
{% block content %}
<section class="panel">
  {% if live.enabled %}
  <div class="live-frame" id="live-frame" data-max-minutes="{{ live.max_view_minutes }}">
    <img id="live-image" alt="Live camera view">
    <div class="live-overlay" id="live-overlay">
      <span id="live-message">Connecting…</span>
      <button class="button primary" id="live-resume" type="button" hidden>Resume live view</button>
    </div>
    <span class="live-badge" id="live-badge" hidden>● LIVE</span>
  </div>
  <div class="player-actions">
    <button class="button" type="button" id="live-toggle">Pause</button>
    <button class="button" type="button" id="live-fullscreen">Full screen</button>
    <span class="muted">Up to {{ live.max_fps|round|int }} frames per second from the low-resolution stream.
      Pauses when this tab is hidden and after {{ live.max_view_minutes }} minutes, so the camera only
      encodes while someone is watching.</span>
  </div>
  {% else %}
  <p class="muted">Live view is turned off (<code>live.enabled</code> is false).</p>
  {% endif %}
</section>
{% endblock %}
EOF

cat > ~/surveillance/web/static/live.js <<'EOF'
"use strict";

// Live MJPEG view. The stream is closed whenever it is not being watched (tab hidden, paused,
// or after max_view_minutes), and the recorder stops encoding once no viewer is connected.
(() => {
  const img = document.getElementById("live-image");
  if (!img) return;
  const frame = document.getElementById("live-frame");
  const overlay = document.getElementById("live-overlay");
  const message = document.getElementById("live-message");
  const resume = document.getElementById("live-resume");
  const badge = document.getElementById("live-badge");
  const toggle = document.getElementById("live-toggle");
  const fullscreen = document.getElementById("live-fullscreen");
  const maxMs = Number(frame.dataset.maxMinutes || 10) * 60000;

  let running = false;
  let pausedByUser = false;
  let retryTimer = null;
  let stopTimer = null;

  function show(text, offerResume) {
    overlay.hidden = false;
    message.textContent = text;
    resume.hidden = !offerResume;
    badge.hidden = true;
  }

  function start() {
    clearTimeout(retryTimer);
    clearTimeout(stopTimer);
    running = true;
    show("Connecting…", false);
    img.src = `/live/stream?ts=${Date.now()}`;
    stopTimer = setTimeout(() => stop("Live view paused to save bandwidth.", true), maxMs);
    toggle.textContent = "Pause";
  }

  function stop(text, offerResume) {
    running = false;
    clearTimeout(retryTimer);
    clearTimeout(stopTimer);
    img.src = "data:,";   // closes the stream connection
    show(text, offerResume);
    toggle.textContent = "Resume";
  }

  img.addEventListener("load", () => {
    if (!running) return;
    overlay.hidden = true;
    badge.hidden = false;
  });
  img.addEventListener("error", () => {
    if (!running) return;
    show("Camera not reachable (recorder stopped, or too many viewers). Retrying…", false);
    retryTimer = setTimeout(start, 3000);
  });

  resume.addEventListener("click", () => { pausedByUser = false; start(); });
  toggle.addEventListener("click", () => {
    if (running) { pausedByUser = true; stop("Paused.", true); } else { pausedByUser = false; start(); }
  });
  fullscreen.addEventListener("click", () => {
    if (document.fullscreenElement) document.exitFullscreen();
    else if (frame.requestFullscreen) frame.requestFullscreen();
  });
  document.addEventListener("visibilitychange", () => {
    if (document.hidden) {
      if (running) stop("Paused while the tab was hidden.", true);
    } else if (!pausedByUser) {
      start();
    }
  });

  start();
})();
EOF
```

## Test procedure

**0. Start both programs again:** `python3 -m app.main` in window 1 and `python3 -m app.web_main` in window 2. Keep the laptop tunnel open. Window 1 should show `LiveStream: Live view ready (up to 8 frames/s, medium quality, on demand)`.

**1. Dashboard:** it should now show the latest camera image in real colour, refreshing about every 10 s. Click it.

**2. Live View:**
- Within a second or two you should see the **● LIVE** badge, and window 1 should show:
  - `Live view started (hardware JPEG, 7.5 frames/s)`
- **Check the delay:** wave at the camera and watch the screen. It should show up in well under a second.
- Click **Pause**. Window 1 should show `Live view stopped (no viewers)`. Click **Resume** and it starts again.
- **Switch to another browser tab for a few seconds.** The stream should stop (the same log line appears) and restart when you come back.
- Try **Full screen**.

**3. Viewer limit:** open the Live View in **4 tabs**. The 4th should show "Camera not reachable (… too many viewers). Retrying…". Close them all afterwards. As soon as the last one closes, the log should show `Live view stopped`.

**4. Recording is unaffected.** Set 60-second segments and restart the recorder:

```bash
python3 tools/set_setting.py recording.segment_seconds 60
```

Keep the Live View open for about 3 minutes, then check the segments recorded meanwhile:

```bash
python3 tools/phase4_check_segments.py --last 3
```

Every segment should be `[PASS]` with no timestamp gaps. Afterwards, set it back with `python3 tools/set_setting.py recording.segment_seconds 300` and restart the recorder.

**5. CPU while watching.** With the Live View open, run:

```bash
top -b -n 2 -d 60 -p "$(pgrep -f '^python3 -m app.main')" | grep python3
```

The second line is the 60-second average; compare it with the 21.8 % you measured before. I expect roughly **+3–8 %** with the hardware encoder.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Window 1 shows `Hardware JPEG encoder unavailable … using software JPEG` | Live view still works, but uses more CPU. Send me the full line. |
| The picture is green, purple or striped | Send me a screenshot. Colour-data layout is the one thing my fake camera couldn't check. |
| It stays on "Connecting…" | Check window 1 for `Live view ready`, and that both programs run as `ysak`. Then run `ls -l /run/user/$(id -u)/surveillance/`, which should list `live.sock`. |
| The live picture freezes but recording carries on | Send me the last lines of `logs/surveillance.log` and `logs/web.log`. |

**Please send me:**
- a screenshot of the Live View;
- the `Live view started (…)` log line;
- the checker result from step 4;
- the CPU figure from step 5.

Phase 11 then adds **authentication**:
- an admin account created from the command line, with an Argon2id-hashed password;
- secure sessions, CSRF protection and brute-force lockout;
- re-entering your password for sensitive actions;
- the Settings page becoming editable.
