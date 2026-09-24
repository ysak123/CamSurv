Phase 12 is ready as version 0.12.0. After these steps, the camera page opens from your laptop or phone anywhere, at `https://cam01.<your-tailnet>.ts.net`, over Tailscale only. There are no router port forwards and no public IP, and the Pi still has nothing listening on the internet.

**How this works:**

- Tailscale's `tailscale serve` handles HTTPS on the Pi's private tailnet address. It passes requests to the web interface on 127.0.0.1:8080, which still accepts only local connections.
- The certificate comes from Let's Encrypt and renews automatically.
- An access policy lets only your own devices reach the camera, on ports 443 (HTTPS) and 22 (SSH). The camera can't start connections to any of your devices.
- Funnel (Tailscale's public-internet sharing) stays off.

**What changed in the code:**

- **New setting `web.https_hostname`:** the tailnet name is added to the Host allow-list. That address gets HSTS (browsers always use HTTPS for it).
- **Stricter origin check:** the same-origin check on form posts now also compares the scheme, `https` for the tailnet name and `http` for the SSH tunnel.
- **Audit log shows the device:** logins record the tailnet address of the phone or laptop, or 127.0.0.1 for the SSH tunnel. This uses the Tailscale proxy's `X-Forwarded-For` header, trusted only from 127.0.0.1. All other forwarding headers are removed.
- **Status fix from Phase 11b:** after you saved settings, the dashboard could show "STOPPED" for about 2 seconds while the recorder was actually running. It now shows "RESTARTING" and goes back to ONLINE within a fraction of a second.
- **`set_setting.py` message:** it now says which program needs a restart.
- **New checker:** `tools/phase12_tailscale_check.py`.

**How I tested it in the VM.** I couldn't create a real tailnet here, so I built a local HTTPS proxy with the same rewrite rules as Tailscale's own source code, plus a fake `tailscale` command. Over the tailnet-style HTTPS address, all of these passed in a browser:

- login and the session cookie;
- the dashboard API;
- live MJPEG (32 frames in 4 seconds);
- video seeking and download;
- a settings save with re-authentication;
- logout.

The SSH-tunnel path still works. The browser test confirmed three rejections:

- posts from `http://` on the tailnet name, from another site, or with `Origin: null` get 403;
- a wrong Host header gets 400;
- the status API without login gets 401.

The tuning-dropdown test, and the earlier storage and database tests, still pass.

---

## Step 1 — Tailscale account and your devices

1. Sign up at [tailscale.com](https://tailscale.com) with Google, Microsoft or GitHub. **Turn on two-factor authentication for that account first.** Whoever controls it controls your tailnet.
2. On your laptop, install Tailscale from [tailscale.com/download](https://tailscale.com/download) and log in with that account.
3. On your phone, install the Tailscale app and log in with the same account.
4. In the admin console ([login.tailscale.com/admin](https://login.tailscale.com/admin)), go to **Settings → Device management** and turn on **Manually approve new devices**. A stolen login then still can't add a device without your approval.

## Step 2 — Install Tailscale on the Pi

Connect to the Pi over SSH as usual, then check the architecture:

```bash
dpkg --print-architecture
```

If it prints `arm64` (expected), run:

```bash
curl -fsSL https://pkgs.tailscale.com/stable/debian/trixie.noarmor.gpg | sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null
curl -fsSL https://pkgs.tailscale.com/stable/debian/trixie.tailscale-keyring.list | sudo tee /etc/apt/sources.list.d/tailscale.list
sudo apt update && sudo apt install -y tailscale
tailscale version
```

If it prints `armhf` instead, use `raspbian` in place of `debian` in both URLs.

## Step 3 — Join the tailnet with a neutral name

```bash
sudo tailscale up --hostname=cam01
```

It prints `To authenticate, visit: https://login.tailscale.com/a/…`. Open that link on your laptop, then approve the device in the admin console. Then check the connection and turn on automatic updates:

```bash
tailscale status
tailscale ip -4
sudo tailscale set --auto-update
```

Expected: `tailscale status` lists `cam01` and your laptop and phone, and `tailscale ip -4` prints an address starting with `100.`.

I chose `cam01` deliberately. The HTTPS certificate will publish `cam01.<tailnet>.ts.net` in public certificate logs, so the name shouldn't reveal your name or address. Also keep the random tailnet name (for example `tail1a2b3c`) rather than a custom one.

## Step 4 — Access policy: only your devices, only HTTPS and SSH

In the admin console, open **Access controls** and replace the whole policy with the one below. Change `YOUR-LOGIN@example.com` to the address you log in with, then click **Save**.

```json
{
  // Only admins (you) may mark a device as a camera.
  "tagOwners": {
    "tag:camera": ["autogroup:admin"]
  },
  "grants": [
    // Your own laptop/phone may reach each other as before.
    {"src": ["autogroup:member"], "dst": ["autogroup:self"], "ip": ["*"]},
    // Your devices may open the camera's web page and SSH into it. Nothing else.
    {"src": ["autogroup:member"], "dst": ["tag:camera"], "ip": ["tcp:443", "tcp:22"]}
    // No rule has the camera as "src": it cannot start a connection to any of your devices.
  ],
  "tests": [
    {"src": "YOUR-LOGIN@example.com",
     "accept": ["tag:camera:443", "tag:camera:22"],
     "deny": ["tag:camera:80", "tag:camera:8080"]}
  ]
}
```

The `tests` block is checked on every save. If it doesn't hold, Tailscale refuses to save the policy.

## Step 5 — Tag the Pi as a camera

In the admin console, open **Machines**, then click **⋯** next to `cam01` → **Edit ACL tags** → add `tag:camera` → **Save**.

Tagging does two things: the policy above now applies to the Pi, and its device key no longer expires. Otherwise the camera would drop off the tailnet after 180 days.

## Step 6 — Turn on MagicDNS and HTTPS certificates

In the admin console, open **DNS**:

- Make sure **MagicDNS** is on.
- Under **HTTPS Certificates**, click **Enable HTTPS**.

Then print the Pi's full tailnet name:

```bash
tailscale status --json | python3 -c "import json,sys; print(json.load(sys.stdin)['Self']['DNSName'].rstrip('.'))"
```

Expected: something like `cam01.tail1a2b3c.ts.net`. I'll call it **YOUR-NAME** below.

## Step 7 — Update the app to 0.12.0 and set the name

Stop the recorder and the web interface (Ctrl+C in each terminal), then create the update script:

```bash
cd ~/surveillance
cat > update_to_0_12_0.py <<'EOF'
#!/usr/bin/env python3
"""Update the surveillance project from 0.11.2 to 0.12.0 (run from ~/surveillance)."""
import shutil, sys
from pathlib import Path

EDITS = [
    ('app/__init__.py',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.11.2"\n',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.12.0"\n'),
    ('app/auth.py',
     '\n    # ------------------------------------------------------------------ login\n    def authenticate(self, username: str, password: str, user_agent: str) -> tuple[str | None, str | None]:\n        """Return (session_token, None) on success or (None, message for the user)."""\n        name = normalize_username(username)[:64]\n        password = password[:MAX_PASSWORD_LENGTH]\n        now = int(time.time())\n',
     '\n    # ------------------------------------------------------------------ login\n    def authenticate(self, username: str, password: str, user_agent: str,\n                     client: str = "") -> tuple[str | None, str | None]:\n        """Return (session_token, None) on success or (None, message for the user).\n        `client` is the address shown in the audit log (a tailnet IP, or 127.0.0.1 for the SSH tunnel)."""\n        name = normalize_username(username)[:64]\n        source = f"from {client[:45]}" if client else ""\n        password = password[:MAX_PASSWORD_LENGTH]\n        now = int(time.time())\n'),
    ('app/auth.py',
     '                                  (now - GLOBAL_WINDOW_S,)).fetchone()[0]\n            if recent >= GLOBAL_MAX_FAILURES:\n                self.audit(conn, safe_label(name), "login_blocked", "too many failures on all accounts")\n                return None, "Too many failed logins recently. Try again in a few minutes."\n            lock = conn.execute("SELECT * FROM lockouts WHERE username = ?", (name,)).fetchone()\n            if lock and lock["locked_until"] > now:\n                minutes = max(1, round((lock["locked_until"] - now) / 60))\n                self.audit(conn, safe_label(name), "login_blocked", "username locked")\n                return None, f"Too many failed attempts. Try again in about {minutes} minute(s)."\n            user = conn.execute("SELECT * FROM users WHERE username = ?", (name,)).fetchone()\n',
     '                                  (now - GLOBAL_WINDOW_S,)).fetchone()[0]\n            if recent >= GLOBAL_MAX_FAILURES:\n                self.audit(conn, safe_label(name), "login_blocked", f"too many failures on all accounts {source}".strip())\n                return None, "Too many failed logins recently. Try again in a few minutes."\n            lock = conn.execute("SELECT * FROM lockouts WHERE username = ?", (name,)).fetchone()\n            if lock and lock["locked_until"] > now:\n                minutes = max(1, round((lock["locked_until"] - now) / 60))\n                self.audit(conn, safe_label(name), "login_blocked", f"username locked {source}".strip())\n                return None, f"Too many failed attempts. Try again in about {minutes} minute(s)."\n            user = conn.execute("SELECT * FROM users WHERE username = ?", (name,)).fetchone()\n'),
    ('app/auth.py',
     '            if not ok:\n                self._register_failure(conn, name, lock, now)\n                self.audit(conn, safe_label(name), "login_failed")\n                log.warning("Failed login for %r", safe_label(name))\n                return None, "Wrong username or password."\n\n',
     '            if not ok:\n                self._register_failure(conn, name, lock, now)\n                self.audit(conn, safe_label(name), "login_failed", source)\n                log.warning("Failed login for %r %s", safe_label(name), source)\n                return None, "Wrong username or password."\n\n'),
    ('app/auth.py',
     '                             (self._hasher.hash(password), user["id"]))\n            token = self._new_session(conn, user["id"], user_agent)\n            self.audit(conn, name, "login")\n            log.info("User %s logged in", name)\n            return token, None\n\n',
     '                             (self._hasher.hash(password), user["id"]))\n            token = self._new_session(conn, user["id"], user_agent)\n            self.audit(conn, name, "login", source)\n            log.info("User %s logged in %s", name, source)\n            return token, None\n\n'),
    ('app/config.py',
     'CAMERA_NAME_PATTERN = re.compile(r"^[\\w][\\w .,\'()-]{0,39}$")\nTUNING_FILE_PATTERN = re.compile(r"^[a-z0-9_]{1,40}\\.json$")\n\n\n',
     'CAMERA_NAME_PATTERN = re.compile(r"^[\\w][\\w .,\'()-]{0,39}$")\nTUNING_FILE_PATTERN = re.compile(r"^[a-z0-9_]{1,40}\\.json$")\nHOSTNAME_PATTERN = re.compile(r"^(?=.{4,253}$)([a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?\\.)+[a-z]{2,63}$")\n\n\n'),
    ('app/config.py',
     '    port: int = 8080\n    status_refresh_seconds: float = 5.0\n\n    def validate(self) -> None:\n',
     '    port: int = 8080\n    status_refresh_seconds: float = 5.0\n    # Name under which the local HTTPS proxy (tailscale serve) publishes this site inside the\n    # private tailnet, e.g. cam01.tail1234.ts.net. Empty: reachable through the SSH tunnel only.\n    https_hostname: str = ""\n\n    def validate(self) -> None:\n'),
    ('app/config.py',
     '                              "exposed directly; remote access goes through the private VPN proxy")\n        _check_range("web.port", self.port, 1024, 65535)\n        _check_range("web.status_refresh_seconds", self.status_refresh_seconds, 2.0, 60.0)\n\n',
     '                              "exposed directly; remote access goes through the private VPN proxy")\n        _check_range("web.port", self.port, 1024, 65535)\n        if self.https_hostname and not HOSTNAME_PATTERN.match(self.https_hostname):\n            raise ConfigError(f"web.https_hostname must be a full lowercase DNS name such as "\n                              f"cam01.tail1234.ts.net (got {self.https_hostname!r})")\n        _check_range("web.status_refresh_seconds", self.status_refresh_seconds, 2.0, 60.0)\n\n'),
    ('app/main.py',
     'import time  # noqa: E402\nfrom pathlib import Path  # noqa: E402\n\nfrom app import __version__  # noqa: E402\n',
     'import time  # noqa: E402\nfrom pathlib import Path  # noqa: E402\nfrom typing import Callable  # noqa: E402\n\nfrom app import __version__  # noqa: E402\n'),
    ('app/main.py',
     '\n\ndef run(settings: Settings, stop: threading.Event) -> None:\n    clock = ClockMonitor()\n    clock.start()\n',
     '\n\ndef run(settings: Settings, stop: threading.Event, restarting: Callable[[], bool] = lambda: False) -> None:\n    clock = ClockMonitor()\n    clock.start()\n'),
    ('app/main.py',
     '                    backoff = BACKOFF_INITIAL_S\n            if stop.is_set():\n                state.set("stopping")\n            recorder.stop()\n            if not stop.is_set():\n',
     '                    backoff = BACKOFF_INITIAL_S\n            if stop.is_set():\n                state.set(*(("restarting", "Applying new settings") if restarting() else ("stopping",)))\n            recorder.stop()\n            if not stop.is_set():\n'),
    ('app/main.py',
     '    finally:\n        active.clear()\n        state.set("stopped")\n        publisher.stop()\n        storage.stop()\n',
     '    finally:\n        active.clear()\n        state.set(*(("restarting", "Applying new settings") if restarting() else ("stopped",)))\n        publisher.stop()\n        storage.stop()\n'),
    ('app/main.py',
     '            watcher.start()\n            try:\n                run(settings, stop)\n            finally:\n                watcher.stop()\n',
     '            watcher.start()\n            try:\n                run(settings, stop, restarting=lambda: watcher.new_settings is not None and not terminate.is_set())\n            finally:\n                watcher.stop()\n'),
    ('app/status.py',
     '\n    def __init__(self) -> None:\n        self.phase = "starting"      # starting | recording | retrying | stopping | stopped\n        self.message = ""\n\n    def set(self, phase: str, message: str = "") -> None:\n        self.phase = phase\n        self.message = message\n\n\n',
     '\n    def __init__(self) -> None:\n        self.phase = "starting"      # starting | recording | retrying | stopping | restarting | stopped\n        self.message = ""\n        self._listeners: list[Callable[[], None]] = []\n\n    def add_listener(self, listener: Callable[[], None]) -> None:\n        self._listeners.append(listener)\n\n    def set(self, phase: str, message: str = "") -> None:\n        self.phase = phase\n        self.message = message\n        for listener in self._listeners:\n            listener()\n\n\n'),
    ('app/status.py',
     '        self._thread = threading.Thread(target=self._run, name="status-publisher", daemon=True)\n        self._write_failed = False\n\n    # Recorder listeners\n',
     '        self._thread = threading.Thread(target=self._run, name="status-publisher", daemon=True)\n        self._write_failed = False\n        self._write_lock = threading.Lock()\n        state.add_listener(self.publish)\n\n    # Recorder listeners\n'),
    ('app/status.py',
     '    def start(self) -> None:\n        self._path.parent.mkdir(parents=True, exist_ok=True, mode=0o700)\n        self._thread.start()\n\n',
     '    def start(self) -> None:\n        self._path.parent.mkdir(parents=True, exist_ok=True, mode=0o700)\n        self.publish()\n        self._thread.start()\n\n'),
    ('app/status.py',
     '\n    def publish(self) -> None:\n        try:\n            atomic_write_bytes(self._path, json.dumps(self.snapshot()).encode(), mode=0o600)\n            self._write_failed = False\n        except Exception as exc:  # noqa: BLE001 - status is informational; never disturb recording\n            if not self._write_failed:\n                log.error("Cannot write the status file %s: %s", self._path, exc)\n            self._write_failed = True\n\n\n',
     '\n    def publish(self) -> None:\n        with self._write_lock:\n            try:\n                atomic_write_bytes(self._path, json.dumps(self.snapshot()).encode(), mode=0o600)\n                self._write_failed = False\n            except Exception as exc:  # noqa: BLE001 - status is informational; never disturb recording\n                if not self._write_failed:\n                    log.error("Cannot write the status file %s: %s", self._path, exc)\n                self._write_failed = True\n\n\n'),
    ('app/web.py',
     '  * listens on 127.0.0.1 only (enforced by config validation)\n  * Host header allow-list, which defeats DNS-rebinding attacks from other websites\n  * strict Content-Security-Policy: no inline scripts, no external resources\n  * templates auto-escape everything; the JavaScript only ever sets textContent\n',
     '  * listens on 127.0.0.1 only (enforced by config validation)\n  * Host header allow-list, which defeats DNS-rebinding attacks from other websites\n  * remote access only through the tailnet: tailscale serve terminates HTTPS on the tailnet\n    address and forwards to 127.0.0.1 (Phase 12); web.https_hostname names that address\n  * strict Content-Security-Policy: no inline scripts, no external resources\n  * templates auto-escape everything; the JavaScript only ever sets textContent\n'),
    ('app/web.py',
     'from app.status import read_status\nfrom app.system_info import SystemInfo\nfrom app.web_auth import install as install_auth\nfrom app.web_live import create_blueprint as create_live_blueprint\nfrom app.web_settings import create_blueprint as create_settings_blueprint\n',
     'from app.status import read_status\nfrom app.system_info import SystemInfo\nfrom app.web_auth import install as install_auth, request_hostname\nfrom app.web_live import create_blueprint as create_live_blueprint\nfrom app.web_settings import create_blueprint as create_settings_blueprint\n'),
    ('app/web.py',
     'STATIC_DIR = PROJECT_ROOT / "web" / "static"\nALLOWED_HOSTNAMES = {"127.0.0.1", "localhost", "[::1]"}\nBITRATE_SAMPLE = 12\n\n',
     'STATIC_DIR = PROJECT_ROOT / "web" / "static"\nALLOWED_HOSTNAMES = {"127.0.0.1", "localhost", "[::1]"}\nHSTS = "max-age=31536000"\nBITRATE_SAMPLE = 12\n\n'),
    ('app/web.py',
     '    app.register_blueprint(create_live_blueprint(settings))\n    system_info = SystemInfo()\n\n    @app.before_request\n    def check_host() -> None:\n        hostname = request.host.rsplit(":", 1)[0] if not request.host.startswith("[") \\\n            else request.host.split("]")[0] + "]"\n        if hostname not in ALLOWED_HOSTNAMES:\n            abort(400)\n\n',
     '    app.register_blueprint(create_live_blueprint(settings))\n    system_info = SystemInfo()\n    https_hostname = settings.web.https_hostname\n    allowed_hostnames = ALLOWED_HOSTNAMES | ({https_hostname} if https_hostname else set())\n\n    @app.before_request\n    def check_host() -> None:\n        if request_hostname() not in allowed_hostnames:\n            abort(400)\n\n'),
    ('app/web.py',
     '            response.headers["Cache-Control"] = "no-store"\n        response.headers.pop("Server", None)\n        return response\n\n',
     '            response.headers["Cache-Control"] = "no-store"\n        response.headers.pop("Server", None)\n        if https_hostname and request_hostname() == https_hostname:\n            response.headers["Strict-Transport-Security"] = HSTS\n        return response\n\n'),
    ('app/web_auth.py',
     '\n\ndef same_origin() -> bool:\n    for header in ("Origin", "Referer"):\n        value = request.headers.get(header)\n        if value:\n            return value != "null" and urlsplit(value).netloc == request.host\n    return False\n\n',
     '\n\ndef request_hostname() -> str:\n    host = request.host.lower()\n    return host.split("]")[0] + "]" if host.startswith("[") else host.rsplit(":", 1)[0]\n\n\ndef same_origin(https_hostname: str) -> bool:\n    """The page that sent this request must be this site, including the scheme: https for the\n    tailnet name (the proxy terminates TLS), plain http for the local SSH-tunnel addresses."""\n    scheme = "https" if https_hostname and request_hostname() == https_hostname else "http"\n    for header in ("Origin", "Referer"):\n        value = request.headers.get(header)\n        if value:\n            parts = urlsplit(value)\n            return value != "null" and parts.scheme == scheme and parts.netloc.lower() == request.host.lower()\n    return False\n\n'),
    ('app/web_auth.py',
     '        if token:\n            g.session = store.get_session(token)\n        if request.method not in SAFE_METHODS and not same_origin():\n            abort(403)\n        if request.endpoint in PUBLIC_ENDPOINTS:\n',
     '        if token:\n            g.session = store.get_session(token)\n        if request.method not in SAFE_METHODS and not same_origin(settings.web.https_hostname):\n            abort(403)\n        if request.endpoint in PUBLIC_ENDPOINTS:\n'),
    ('app/web_auth.py',
     '        password = request.form.get("password", "")\n        target = request.form.get("next", "")\n        token, error = store.authenticate(username, password, request.headers.get("User-Agent", ""))\n        if error:\n            return render_template("login.html", title="Log in", page="", next=target, error=error,\n',
     '        password = request.form.get("password", "")\n        target = request.form.get("next", "")\n        token, error = store.authenticate(username, password, request.headers.get("User-Agent", ""),\n                                          request.remote_addr or "")\n        if error:\n            return render_template("login.html", title="Log in", page="", next=target, error=error,\n'),
    ('app/web_main.py',
     'Run from the project directory:   python3 -m app.web_main [--config PATH]\n\nServes the dashboard on 127.0.0.1 only. From your computer, open an SSH tunnel:\n    ssh -L 8080:127.0.0.1:8080 ysak@ysak.local\nthen browse to http://localhost:8080\n"""\nfrom __future__ import annotations\n',
     'Run from the project directory:   python3 -m app.web_main [--config PATH]\n\nServes the dashboard on 127.0.0.1 only. Two ways in, both private:\n  * inside the tailnet: tailscale serve publishes it as https://<web.https_hostname> (Phase 12)\n  * SSH tunnel:  ssh -N -L 8080:127.0.0.1:8080 ysak@ysak.local  then http://localhost:8080\n"""\nfrom __future__ import annotations\n'),
    ('app/web_main.py',
     '# Each live viewer holds one worker thread for as long as it watches; keep spare threads for pages.\nSPARE_WORKER_THREADS = 4\n\n\n',
     '# Each live viewer holds one worker thread for as long as it watches; keep spare threads for pages.\nSPARE_WORKER_THREADS = 4\n# tailscale serve connects from 127.0.0.1 and names the tailnet client in X-Forwarded-For. Only that\n# header is trusted (for the audit log); Waitress removes all other forwarding headers.\nPROXY_ADDRESS = "127.0.0.1"\n\n\n'),
    ('app/web_main.py',
     '    host, port = settings.web.host, settings.web.port\n    log.info("Web interface %s listening on http://%s:%d (local only)", __version__, host, port)\n    serve(create_app(settings, args.config), host=host, port=port, threads=settings.live.max_viewers + SPARE_WORKER_THREADS,\n          ident="", clear_untrusted_proxy_headers=True)\n    return 0\n\n',
     '    host, port = settings.web.host, settings.web.port\n    log.info("Web interface %s listening on http://%s:%d (local only)", __version__, host, port)\n    if settings.web.https_hostname:\n        log.info("Tailnet address: https://%s (through tailscale serve)", settings.web.https_hostname)\n    serve(create_app(settings, args.config), host=host, port=port, threads=settings.live.max_viewers + SPARE_WORKER_THREADS,\n          ident="", trusted_proxy=PROXY_ADDRESS, trusted_proxy_count=1, trusted_proxy_headers={"x-forwarded-for"},\n          clear_untrusted_proxy_headers=True)\n    return 0\n\n'),
    ('app/web_settings.py',
     '            fixed={"Recordings folder": current.paths.recordings_dir, "Database folder": current.paths.database_dir,\n                   "Web address": f"{current.web.host}:{current.web.port}",\n                   "Required mount": current.storage.required_mount or "none"}), status\n\n',
     '            fixed={"Recordings folder": current.paths.recordings_dir, "Database folder": current.paths.database_dir,\n                   "Web address": f"{current.web.host}:{current.web.port}",\n                   "Tailnet address": f"https://{current.web.https_hostname}" if current.web.https_hostname\n                   else "not set (SSH tunnel only)",\n                   "Required mount": current.storage.required_mount or "none"}), status\n\n'),
    ('tools/set_setting.py',
     '\nValues are parsed as JSON when possible (numbers, true/false), otherwise used as text.\nRestart the recorder afterwards for the change to take effect.\n"""\nfrom __future__ import annotations\n',
     '\nValues are parsed as JSON when possible (numbers, true/false), otherwise used as text.\nA running recorder reloads its settings by itself; web.* and auth.* changes need the web\ninterface restarted.\n"""\nfrom __future__ import annotations\n'),
    ('tools/set_setting.py',
     '        print(f"Not saved: {exc}", file=sys.stderr)\n        return 2\n    print(f"{args.key}: {old!r} -> {value!r}  (restart the recorder to apply)")\n    return 0\n\n',
     '        print(f"Not saved: {exc}", file=sys.stderr)\n        return 2\n    note = ("restart the web interface to apply" if section in ("web", "auth")\n            else "a running recorder applies it within a few seconds")\n    print(f"{args.key}: {old!r} -> {value!r}  ({note})")\n    return 0\n\n'),
    ('web/static/app.js',
     '      retrying: ["OFFLINE", "bad", rec.recorder.message],\n      stopping: ["STOPPING", "warn", ""],\n      stopped: ["STOPPED", "bad", "The recorder was stopped."],\n    };\n',
     '      retrying: ["OFFLINE", "bad", rec.recorder.message],\n      stopping: ["STOPPING", "warn", ""],\n      restarting: ["RESTARTING", "warn", rec.recorder.message],\n      stopped: ["STOPPED", "bad", "The recorder was stopped."],\n    };\n'),
]

texts = {}
for rel, old, new in EDITS:
    text = texts.setdefault(rel, Path(rel).read_text())
    if text.count(old) != 1:
        sys.exit(f"ABORTED, nothing changed: {rel} does not match the expected 0.11.2 code "
                 f"(found {text.count(old)} matches for:\n{old})")
    texts[rel] = text.replace(old, new)
for rel, text in texts.items():
    shutil.copy2(rel, rel + ".bak")
    Path(rel).write_text(text)
    print(f"updated {rel}  (backup: {rel}.bak)")
print("Update to 0.12.0 complete.")
EOF
python3 update_to_0_12_0.py
```

Then create the checker, a new file:

```bash
cat > ~/surveillance/tools/phase12_tailscale_check.py <<'EOF'
#!/usr/bin/env python3
"""Phase 12: check that the camera is reachable only through the tailnet, over HTTPS.

    python3 ~/surveillance/tools/phase12_tailscale_check.py

Read-only. Checks the Tailscale connection, the HTTPS certificate, the `tailscale serve`
mapping to the local web interface, that Funnel (public internet sharing) is off, which
ports the Pi listens on, and finally fetches the login page through the tailnet address.
"""
from __future__ import annotations

import argparse
import ipaddress
import json
import shutil
import socket
import ssl
import subprocess
import sys
import time
from datetime import datetime, timezone
from pathlib import Path
from urllib.parse import urlsplit

sys.path.insert(0, str(Path(__file__).resolve().parent.parent))

from app.config import DEFAULT_CONFIG_PATH, ConfigError, load_settings  # noqa: E402

TAILNET_V4 = ipaddress.ip_network("100.64.0.0/10")
TAILNET_V6 = ipaddress.ip_network("fd7a:115c:a1e0::/48")
CAMERA_TAG = "tag:camera"
HTTPS_TIMEOUT_S = 60

results: list[str] = []


def report(status: str, text: str) -> None:
    results.append(status)
    print(f"  [{status}] {text}")


def run_json(cmd: list[str]) -> dict | None:
    try:
        out = subprocess.run(cmd, capture_output=True, text=True, timeout=20)
    except (OSError, subprocess.TimeoutExpired) as exc:
        report("FAIL", f"{' '.join(cmd)} failed: {exc}")
        return None
    if out.returncode != 0:
        report("FAIL", f"{' '.join(cmd)} failed: {(out.stderr or out.stdout).strip()[:300]}")
        return None
    try:
        return json.loads(out.stdout or "{}")
    except ValueError:
        report("FAIL", f"{' '.join(cmd)} did not return JSON")
        return None


def is_loopback(addr: str) -> bool:
    try:
        return ipaddress.ip_address(addr).is_loopback
    except ValueError:
        return addr == "localhost"


def is_tailnet(addr: str) -> bool:
    try:
        ip = ipaddress.ip_address(addr)
    except ValueError:
        return False
    return ip in (TAILNET_V4 if ip.version == 4 else TAILNET_V6)


def listening_tcp() -> list[tuple[str, int, str]]:
    """(address, port, process) of every listening TCP socket."""
    try:
        out = subprocess.run(["ss", "-Hltnp"], capture_output=True, text=True, timeout=10).stdout
    except (OSError, subprocess.TimeoutExpired):
        return []
    sockets = []
    for line in out.splitlines():
        parts = line.split()
        if len(parts) < 4:
            continue
        local = parts[3]
        addr, _, port = local.rpartition(":")
        addr = addr.strip("[]").split("%")[0]
        process = ""
        if "users:((" in line:
            process = line.split('users:(("', 1)[1].split('"', 1)[0]
        if port.isdigit():
            sockets.append((addr, int(port), process))
    return sockets


def check_https(dns_name: str, ip: str, cafile: str | None) -> None:
    context = ssl.create_default_context(cafile=cafile)
    deadline = time.monotonic() + HTTPS_TIMEOUT_S
    while True:
        try:
            with socket.create_connection((ip, 443), timeout=15) as raw:
                with context.wrap_socket(raw, server_hostname=dns_name) as tls:
                    cert = tls.getpeercert()
                    responses = {}
                    for path in ("/login", "/api/status"):
                        tls.sendall(f"GET {path} HTTP/1.1\r\nHost: {dns_name}\r\nUser-Agent: phase12-check\r\n"
                                    f"Connection: {'close' if path == '/api/status' else 'keep-alive'}\r\n\r\n"
                                    .encode())
                        responses[path] = read_response(tls)
            break
        except ssl.SSLCertVerificationError as exc:
            report("FAIL", f"HTTPS certificate for {dns_name} is not valid: {exc.verify_message}")
            return
        except (OSError, ssl.SSLError) as exc:
            if time.monotonic() > deadline:
                report("FAIL", f"cannot reach https://{dns_name} ({ip}:443): {exc}")
                return
            print(f"         waiting for HTTPS on {ip}:443 ({exc}); the first certificate can take a minute ...")
            time.sleep(5)
    expires = datetime.strptime(cert["notAfter"], "%b %d %H:%M:%S %Y %Z").replace(tzinfo=timezone.utc)
    days = (expires - datetime.now(timezone.utc)).days
    report("PASS", f"HTTPS certificate valid for {dns_name} (issued by "
                   f"{dict(x[0] for x in cert['issuer']).get('organizationName', '?')}, renews automatically, "
                   f"{days} days left)")
    status, headers = responses["/login"]
    if status == 200:
        report("PASS", f"https://{dns_name}/login answers 200 through tailscale serve")
    elif status == 400:
        report("FAIL", f"https://{dns_name}/login answers 400: the web interface does not accept this name. "
                       f"Set web.https_hostname (step 7) and restart the web interface")
    else:
        report("FAIL", f"https://{dns_name}/login answers {status}")
    if "strict-transport-security" in headers:
        report("PASS", "browsers are told to always use HTTPS for this name (HSTS)")
    elif status == 200:
        report("WARN", "no HSTS header: is the web interface updated to 0.12.0 and restarted?")
    status, _headers = responses["/api/status"]
    report("PASS" if status == 401 else "FAIL",
           f"https://{dns_name}/api/status without logging in answers {status} (expected 401)")


def read_response(tls: ssl.SSLSocket) -> tuple[int, dict[str, str]]:
    data = b""
    while b"\r\n\r\n" not in data:
        chunk = tls.recv(4096)
        if not chunk:
            break
        data += chunk
    head, _, body = data.partition(b"\r\n\r\n")
    lines = head.decode("latin-1").split("\r\n")
    status = int(lines[0].split()[1]) if lines and len(lines[0].split()) > 1 else 0
    headers = {k.strip().lower(): v.strip() for k, _, v in (line.partition(":") for line in lines[1:])}
    length = int(headers.get("content-length", "0") or 0)
    while len(body) < length:
        chunk = tls.recv(4096)
        if not chunk:
            break
        body += chunk
    return status, headers


def main() -> int:
    parser = argparse.ArgumentParser(description="Phase 12: Tailscale / HTTPS check (read-only)")
    parser.add_argument("--config", type=Path, default=DEFAULT_CONFIG_PATH)
    parser.add_argument("--tailscale", default="tailscale", help=argparse.SUPPRESS)
    parser.add_argument("--cafile", help=argparse.SUPPRESS)
    args = parser.parse_args()
    try:
        settings, _ = load_settings(args.config, create_if_missing=False)
    except ConfigError as exc:
        print(f"Configuration error: {exc}", file=sys.stderr)
        return 2
    web_port = settings.web.port

    print("Tailscale")
    if not shutil.which(args.tailscale):
        report("FAIL", "tailscale is not installed (step 2)")
        return finish()
    status = run_json([args.tailscale, "status", "--json"])
    if status is None:
        return finish()
    report("PASS" if status.get("BackendState") == "Running" else "FAIL",
           f"Tailscale state: {status.get('BackendState')} (version {status.get('Version', '?').split('-')[0]})")
    me = status.get("Self") or {}
    dns_name = (me.get("DNSName") or "").rstrip(".").lower()
    ips = [ip for ip in me.get("TailscaleIPs") or [] if ":" not in ip]
    tailnet = status.get("CurrentTailnet") or {}
    print(f"         this Pi in the tailnet: {dns_name or '?'}  {' '.join(me.get('TailscaleIPs') or [])}")
    if not dns_name or not ips:
        report("FAIL", "no tailnet name or address yet: finish `sudo tailscale up` (step 3)")
        return finish()
    report("PASS" if tailnet.get("MagicDNSEnabled") else "FAIL",
           "MagicDNS " + ("on" if tailnet.get("MagicDNSEnabled") else "off: turn it on in the admin console, DNS page"))
    tags = list(me.get("Tags") or [])
    report("PASS" if CAMERA_TAG in tags else "WARN",
           f"device tags: {', '.join(tags) or 'none'}" + ("" if CAMERA_TAG in tags else
                                                          f" (add {CAMERA_TAG} in the admin console, step 5)"))
    expiry = me.get("KeyExpiry")
    if expiry:
        report("WARN", f"device key expires {expiry[:10]}: the camera would drop off the tailnet then "
                       f"(tagging it disables expiry)")
    else:
        report("PASS", "device key does not expire")
    cert_domains = [d.rstrip(".").lower() for d in status.get("CertDomains") or []]
    report("PASS" if dns_name in cert_domains else "FAIL",
           "HTTPS certificates " + ("enabled" if dns_name in cert_domains
                                    else "not enabled: admin console, DNS page, Enable HTTPS (step 6)"))

    print("tailscale serve")
    serve = run_json([args.tailscale, "serve", "status", "--json"]) or {}
    handler = (((serve.get("Web") or {}).get(f"{dns_name}:443") or {}).get("Handlers") or {}).get("/") or {}
    proxy = handler.get("Proxy", "")
    target = urlsplit(proxy if "://" in proxy else f"http://{proxy}") if proxy else None
    if target and target.scheme == "http" and target.hostname in ("127.0.0.1", "localhost") \
            and target.port == web_port and (serve.get("TCP") or {}).get("443", {}).get("HTTPS"):
        report("PASS", f"https://{dns_name} -> {proxy}")
    else:
        report("FAIL", f"https://{dns_name}/ is not forwarded to http://127.0.0.1:{web_port} "
                       f"(found {proxy or 'nothing'}); run the serve command in step 8")
    funnel = [hp for hp, on in (serve.get("AllowFunnel") or {}).items() if on]
    if funnel:
        report("FAIL", f"Funnel is ON for {', '.join(funnel)}: the camera is on the public internet! "
                       f"Run: sudo tailscale funnel reset")
    else:
        report("PASS", "Funnel is off: nothing is shared outside the tailnet")
    others = sorted(p for p in (serve.get("TCP") or {}) if p != "443")
    if others:
        report("WARN", f"tailscale serve also publishes port(s) {', '.join(others)}")

    print("Web interface")
    expected = settings.web.https_hostname
    report("PASS" if expected == dns_name else "FAIL",
           f"web.https_hostname = {expected or '(empty)'}" + ("" if expected == dns_name else
           f"; run: python3 ~/surveillance/tools/set_setting.py web.https_hostname {dns_name}  (then restart it)"))

    print("Listening ports")
    sockets = listening_tcp()
    web = [s for s in sockets if s[1] == web_port]
    if web and all(is_loopback(a) for a, _p, _n in web):
        report("PASS", f"web interface port {web_port} listens on this Pi only (127.0.0.1)")
    elif web:
        report("FAIL", f"port {web_port} is reachable from the network: {web}")
    else:
        report("WARN", f"nothing listens on port {web_port}: is the web interface running?")
    for addr, port, name in sorted(set(sockets)):
        if is_loopback(addr) or port == web_port:
            continue
        who = f" ({name})" if name else ""
        if is_tailnet(addr):
            print(f"         tailnet only : {addr}:{port}{who}")
        else:
            label = "SSH, restricted to the tailnet in Phase 14" if port == 22 else "check what this is"
            report("WARN", f"reachable from the home network: {addr or '*'}:{port}{who}: {label}")

    print("End-to-end HTTPS through the tailnet")
    check_https(dns_name, ips[0], args.cafile)
    return finish()


def finish() -> int:
    fails, warns = results.count("FAIL"), results.count("WARN")
    verdict = "TAILNET ACCESS NOT READY" if fails else "TAILNET ACCESS OK"
    print(f"\nRESULT: {verdict}  ({results.count('PASS')} pass, {warns} warn, {fails} fail)")
    return 1 if fails else 0


if __name__ == "__main__":
    sys.exit(main())
EOF
cd ~/surveillance && sha256sum app/__init__.py app/auth.py app/config.py app/main.py app/status.py app/web.py app/web_auth.py app/web_main.py app/web_settings.py tools/set_setting.py web/static/app.js tools/phase12_tailscale_check.py
```

Expected checksums:

```
1efa1678f66b2b0ed34dfdf16512f8fa61c2885e4ec98a2fe72d84a498c6852d  app/__init__.py
998c6a9c73cd4e5b64a5ac0239351a307e0ec9e02d57837ad32243d4ee4236af  app/auth.py
e7d31faf82f80a97a06f9a2ebbd342c919dc5c8b9c020acd5eb44f7d6133de44  app/config.py
714ba5f4f8f70b08e161e6007a642c85c427d5f620772ff393b9fd0f19aadd62  app/main.py
ae7b0d98e0a2ab74a481647c9f82e63e2c2bfaefa4337abfacd5737040635355  app/status.py
5b0c272c7b5f0f1b997ed9e73188103a58da3bcb1e6d492905823a816634bdef  app/web.py
0341425f0be9009e6f4d218ac446f1d5a27b8bbd86d97b9d04a49932d1cfed3c  app/web_auth.py
b2873a18d7a399851cd0a7cf1a27a89fadb9dd29d5cf5b8b890bc44872742185  app/web_main.py
cef58b84b320e8e2c0871104cc7d679d6b067a7e5dc8dba0fb12915ed7981758  app/web_settings.py
6541409bf57715b98c716e361668a5fe7395cf78bd23cb07fb12de0572589272  tools/set_setting.py
db63f25f64ac7ed1c2cd11c9a285d28b8f754e18cebc17c150cc2505a8b91e36  web/static/app.js
636128cf627449f02a9e6374b90f15ad7491fe2af3d1c562f640f1b9a36dc694  tools/phase12_tailscale_check.py
```

Now set the tailnet name (use your real YOUR-NAME), then start the recorder and the web interface as before:

```bash
python3 ~/surveillance/tools/set_setting.py web.https_hostname YOUR-NAME
```

Expected output: `web.https_hostname: '' -> 'YOUR-NAME'  (restart the web interface to apply)`. When the web interface starts, its log should include `Tailnet address: https://YOUR-NAME (through tailscale serve)`.

## Step 8 — Publish the page inside the tailnet (never Funnel)

```bash
sudo tailscale serve --bg --https=443 http://127.0.0.1:8080
tailscale serve status
```

Expected: `https://YOUR-NAME (tailnet only)` with `|-- / proxy http://127.0.0.1:8080`. The `--bg` setting survives reboots.

Never run `tailscale funnel`, which would put the camera on the public internet. To undo this step: `sudo tailscale serve --https=443 off`.

## Step 9 — Run the checker on the Pi

```bash
python3 ~/surveillance/tools/phase12_tailscale_check.py
```

Expected: everything PASS, ending with `RESULT: TAILNET ACCESS OK`.

- The first run may wait up to a minute while the certificate is issued.
- A WARN for SSH on `0.0.0.0:22` / `:::22` is expected. Phase 14 limits SSH to the tailnet.
- Tell me about any other "reachable from the home network" lines.

## Step 10 — Laptop test

With Tailscale on, open `https://YOUR-NAME` in your browser. There's no SSH tunnel needed now, and no certificate warning should appear. Then:

1. Log in. It's a new address, so it's a new session.
2. Check the Dashboard, Live View, and playing and downloading a recording.
3. Save one harmless setting (for example, change a cooldown and then change it back).
4. On the Account page, the latest "login" line should say `from 100.x.y.z` (your laptop's tailnet address).

Also check SSH over the tailnet from the laptop:

```bash
ssh ysak@cam01
```

The old tunnel (`ssh -N -L 8080:127.0.0.1:8080 ysak@ysak.local` → http://localhost:8080) still works as a fallback.

## Step 11 — Phone away from home

1. Turn Wi-Fi **off** so the phone uses mobile data.
2. Turn Tailscale on and open `https://YOUR-NAME`.
3. Log in, open Live View, and play a motion event.
4. Now turn the Tailscale app **off** and reload. The page should fail to load, because the name doesn't exist outside your tailnet.

Safari logins, which failed on http://localhost earlier, work here because the address is HTTPS.

## Step 12 — Confirm what's blocked, and measure the load

On the Pi, pinging your laptop should get no replies (the camera may not start connections):

```bash
ping -c 3 <laptop tailnet IP from `tailscale status`>
```

Expected: `100% packet loss`. A normal `ping cam01` from the laptop also fails by design. Only TCP 443 and 22 are allowed.

Then, with Live View open on the phone over the tailnet, run `top -b -n 3 -d 5 | grep -E "python3|tailscaled"` on the Pi.

**What's still true (this isn't 100% security):**

- Anyone who can log in to your Tailscale account, or who controls a device you approved, can reach the login page. They still need the camera password.
- Tailscale's coordination service is trusted to hand out the right keys. Tailnet Lock removes that trust, if you want it later.
- Tailscale may use UPnP on your router to open a UDP port for faster direct connections. That port carries only encrypted WireGuard traffic and answers nothing else.
- The name `cam01.<tailnet>.ts.net` appears in public certificate logs, but it can't be reached from the internet.

Please send me:
1. The checksum output from Step 7.
2. The output of `tailscale serve status`.
3. The full checker output from Step 9.
4. Whether the laptop, phone-on-mobile-data and "Tailscale off" tests behaved as described.
5. The ping result and the `top` lines from Step 12.

Once this works, Phase 13 turns the recorder and web interface into systemd services that start at boot and restart themselves.
