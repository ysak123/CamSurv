Phase 13b is ready as version 0.13.2. The Settings page now has a Time zone section and Restart / Shut down buttons at the bottom.

**How it's protected:**

- **Fixed actions only:** the browser can choose one of three fixed actions and nothing else. The time zone must be one of the system's own zone names, and no command is ever built from browser input.
- **Password every time:** Restart and Shut down always ask for your password, and the browser shows a confirmation dialog first. The time zone uses the same 5-minute password window as other settings.
- **Normal protections still apply:** CSRF token and same-origin checks, lockout counting, and an entry in the audit log.
- **Narrow permission rule:** a polkit rule, the operating system's permission system, allows exactly these three actions. It allows them only for processes of user `cctv` inside `surveillance-web.service`. The recorder runs as the same user and is still refused.
- **Recorder log times:** after a time-zone change, the recorder's log timestamps switch to the new zone without a restart.

I tested it on a real Debian 13 system running systemd in a container:

- **Refused callers:** any other unit running as `cctv` is refused ("Interactive authentication required", and the reboot check answers "challenge").
- **Rejected requests:** a wrong password, a request without the CSRF token, and an unknown action were all rejected.
- **Working actions:** from the website, a time-zone change, Restart and Shut down all worked, and the audit log recorded each one with your account name.
- **Clean stops:** on both restart and shutdown, the recorder finished its segment cleanly first.

## Step 1 — Apply the update

```bash
cd ~/surveillance
cat > update_to_0_13_2.py <<'PYEOF'
#!/usr/bin/env python3
"""Update the surveillance project from 0.13.1 to 0.13.2 (run from ~/surveillance)."""
import os, shutil, sys
from pathlib import Path

EDITS = [
    ('app/__init__.py',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.13.1"\n',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.13.2"\n'),
    ('app/main.py',
     '    def _run(self) -> None:\n        while not self._halt.wait(SETTINGS_CHECK_S):\n            signature = self._file_signature()\n            if signature is None or signature == self._signature:\n',
     '    def _run(self) -> None:\n        while not self._halt.wait(SETTINGS_CHECK_S):\n            time.tzset()  # picks up a time zone change (Settings page) for the log timestamps\n            signature = self._file_signature()\n            if signature is None or signature == self._signature:\n'),
    ('app/web_settings.py',
     '  and this web process restarts itself so it uses the new values too.\n* Paths, network address and log file details are deliberately not editable from the browser.\n"""\nfrom __future__ import annotations\n',
     '  and this web process restarts itself so it uses the new values too.\n* Paths, network address and log file details are deliberately not editable from the browser.\n* Phase 13b: the time zone, and Restart / Shut down (password needed every time), through the\n  fixed actions in app/system_control.py.\n"""\nfrom __future__ import annotations\n'),
    ('app/web_settings.py',
     'from pathlib import Path\n\nfrom flask import Blueprint, g, render_template, request\n\nfrom app.auth import AuthError, AuthStore\n',
     'from pathlib import Path\n\nfrom flask import Blueprint, abort, g, render_template, request\n\nfrom app.auth import AuthError, AuthStore\n'),
    ('app/web_settings.py',
     '                        ConfigError, Settings, diff_settings, load_settings, save_settings, settings_from_dict)\nfrom app.status import read_status\n\nlog = logging.getLogger("Settings")\n',
     '                        ConfigError, Settings, diff_settings, load_settings, save_settings, settings_from_dict)\nfrom app.status import read_status\nfrom app.system_control import (POWER_ACTIONS, SystemControlError, check_power, current_time_zone, schedule_power,\n                                set_time_zone, time_zones)\n\nlog = logging.getLogger("Settings")\n'),
    ('app/web_settings.py',
     'REAUTH_MAX_AGE_S = 300\nRESTART_DELAY_S = 1.5\nRESOLUTIONS = {\n    "1296x972": (1296, 972, 640, 480, "1296 × 972 – full field of view (recommended for OV5647)"),\n',
     'REAUTH_MAX_AGE_S = 300\nRESTART_DELAY_S = 1.5\nPOWER_DELAY_S = 3.0\nRESOLUTIONS = {\n    "1296x972": (1296, 972, 640, 480, "1296 × 972 – full field of view (recommended for OV5647)"),\n'),
    ('app/web_settings.py',
     '    bp = Blueprint("cfg", __name__)\n    restart_lock = threading.Lock()\n\n    def current_settings() -> tuple[Settings, str | None]:\n',
     '    bp = Blueprint("cfg", __name__)\n    restart_lock = threading.Lock()\n    zones = time_zones()\n\n    def restart_soon() -> None:\n        if restart_lock.acquire(blocking=False):\n            threading.Timer(RESTART_DELAY_S, restart_web_process).start()\n\n    def current_settings() -> tuple[Settings, str | None]:\n'),
    ('app/web_settings.py',
     '            errors=list(errors), message=message, changes=list(changes), reloading=reloading,\n            reauth_needed=not store.reauth_fresh(g.session, REAUTH_MAX_AGE_S),\n            fixed={"Recordings folder": current.paths.recordings_dir, "Database folder": current.paths.database_dir,\n                   "Web address": f"{current.web.host}:{current.web.port}",\n',
     '            errors=list(errors), message=message, changes=list(changes), reloading=reloading,\n            reauth_needed=not store.reauth_fresh(g.session, REAUTH_MAX_AGE_S),\n            time_zones=zones, time_zone=current_time_zone(),\n            fixed={"Recordings folder": current.paths.recordings_dir, "Database folder": current.paths.database_dir,\n                   "Web address": f"{current.web.host}:{current.web.port}",\n'),
    ('app/web_settings.py',
     '        store.record(g.session["username"], "settings_changed", summary)\n        log.info("Settings changed by %s: %s", g.session["username"], summary)\n        if restart_lock.acquire(blocking=False):\n            threading.Timer(RESTART_DELAY_S, restart_web_process).start()\n        return render(asdict(new), new, message="Settings saved.", changes=changes, reloading=True)\n\n    return bp\n',
     '        store.record(g.session["username"], "settings_changed", summary)\n        log.info("Settings changed by %s: %s", g.session["username"], summary)\n        restart_soon()\n        return render(asdict(new), new, message="Settings saved.", changes=changes, reloading=True)\n\n    @bp.post("/settings/timezone")\n    def save_time_zone():\n        current, _problem = current_settings()\n        zone = request.form.get("timezone", "")\n        if zone == current_time_zone():\n            return render(asdict(current), current, message="Nothing changed.")\n        try:\n            if not store.reauth_fresh(g.session, REAUTH_MAX_AGE_S):\n                store.reauth(g.session, request.form.get("current_password", ""))\n            set_time_zone(zone, zones)\n        except (AuthError, SystemControlError) as exc:\n            return render(asdict(current), current, errors=[str(exc)], status=403)\n        store.record(g.session["username"], "timezone_changed", zone)\n        log.info("Time zone changed to %s by %s", zone, g.session["username"])\n        restart_soon()\n        return render(asdict(current), current, message=f"Time zone set to {zone}.", reloading=True)\n\n    @bp.post("/system/power")\n    def power():\n        action = request.form.get("action", "")\n        if action not in POWER_ACTIONS:\n            abort(400)\n        current, _problem = current_settings()\n        try:\n            store.reauth(g.session, request.form.get("power_password", ""))\n            check_power(action)\n        except (AuthError, SystemControlError) as exc:\n            return render(asdict(current), current, errors=[str(exc)], status=403)\n        store.record(g.session["username"], action, "requested from the web interface")\n        log.warning("%s requested by %s from the web interface", action, g.session["username"])\n        schedule_power(action, POWER_DELAY_S)\n        return render_template("power.html", title="Restarting" if action == "reboot" else "Shutting down",\n                               page="cfg.settings_page", action=action)\n\n    return bp\n'),
    ('web/templates/settings.html',
     '  <strong>{{ message }}</strong>\n  {% if changes %}<ul>{% for key, old, new in changes %}<li><code>{{ key }}</code>: {{ old }} &rarr; {{ new }}</li>{% endfor %}</ul>{% endif %}\n  {% if reloading %}<p>The web interface restarts now. The recorder applies recording changes within about\n    10 seconds; a few seconds of video are skipped while the camera restarts. This page reloads by itself.</p>{% endif %}\n</div>\n{% endif %}\n',
     '  <strong>{{ message }}</strong>\n  {% if changes %}<ul>{% for key, old, new in changes %}<li><code>{{ key }}</code>: {{ old }} &rarr; {{ new }}</li>{% endfor %}</ul>{% endif %}\n  {% if reloading %}<p>The web interface restarts now{% if changes %}. The recorder applies recording changes within\n    about 10 seconds; a few seconds of video are skipped while the camera restarts{% endif %}. This page reloads by\n    itself.</p>{% endif %}\n</div>\n{% endif %}\n'),
    ('web/templates/settings.html',
     '    <section class="panel">\n      <h2>Not editable here</h2>\n      <p class="help">These are changed only on the Pi with <code>tools/set_setting.py</code>, because a mistake\n        could lock you out or send recordings to the wrong disk.</p>\n      <table class="kv">\n        {% for k, v in fixed.items() %}<tr><th scope="row">{{ k }}</th><td>{{ v }}</td></tr>{% endfor %}\n',
     '    <section class="panel">\n      <h2>Not editable here</h2>\n      <p class="help">These are changed only on the Pi with <code>sudo cctv-tool set_setting</code>, because a\n        mistake could lock you out or send recordings to the wrong disk.</p>\n      <table class="kv">\n        {% for k, v in fixed.items() %}<tr><th scope="row">{{ k }}</th><td>{{ v }}</td></tr>{% endfor %}\n'),
    ('web/templates/settings.html',
     '  </section>\n</form>\n{% endblock %}\n',
     '  </section>\n</form>\n\n<div class="settings-grid system-grid">\n  <section class="panel">\n    <h2>Time zone</h2>\n    <form method="post" action="{{ url_for(\'cfg.save_time_zone\') }}">\n      <input type="hidden" name="csrf_token" value="{{ csrf_token }}">\n      <div class="field">\n        <label for="timezone">Used for the times shown on these pages and in the logs</label>\n        <select id="timezone" name="timezone">\n          {% if time_zone not in time_zones %}<option value="{{ time_zone or \'\' }}" selected>{{ time_zone or "unknown" }}</option>{% endif %}\n          {% for zone in time_zones %}\n            <option value="{{ zone }}" {% if zone == time_zone %}selected{% endif %}>{{ zone.replace("_", " ") }}</option>\n          {% endfor %}\n        </select>\n        <p class="help">Recordings are always named and stored in UTC, so changing this never mixes them up.</p>\n      </div>\n      {% if reauth_needed %}\n      <div class="field">\n        <label for="tz-password">Your password</label>\n        <input id="tz-password" type="password" name="current_password" autocomplete="current-password" maxlength="256" required>\n      </div>\n      {% endif %}\n      <button class="button primary" type="submit">Set time zone</button>\n    </form>\n  </section>\n  <section class="panel">\n    <h2>Restart or shut down the Pi</h2>\n    <form method="post" action="{{ url_for(\'cfg.power\') }}">\n      <input type="hidden" name="csrf_token" value="{{ csrf_token }}">\n      <div class="field">\n        <label for="power-password">Your password (needed every time)</label>\n        <input id="power-password" type="password" name="power_password" autocomplete="current-password" maxlength="256" required>\n        <p class="help">Recording stops while the Pi restarts (about a minute). After a shutdown the Pi stays off\n          until its power is unplugged and plugged back in.</p>\n      </div>\n      <div class="button-row">\n        <button class="button" type="submit" name="action" value="reboot"\n                data-confirm="Restart the Pi now? Recording stops for about a minute.">Restart Pi</button>\n        <button class="button danger" type="submit" name="action" value="poweroff"\n                data-confirm="Shut the Pi down? It will NOT start again until its power is unplugged and plugged back in.">Shut down Pi</button>\n      </div>\n    </form>\n  </section>\n</div>\n{% endblock %}\n'),
    ('web/static/settings.js',
     '  setTimeout(() => { window.location.href = "/settings"; }, Number(notice.dataset.seconds || 6) * 1000);\n})();\n',
     '  setTimeout(() => { window.location.href = "/settings"; }, Number(notice.dataset.seconds || 6) * 1000);\n})();\n\n// Restart / Shut down ask once more before the form is sent.\ndocument.querySelectorAll("button[data-confirm]").forEach((button) => {\n  button.addEventListener("click", (event) => {\n    if (!window.confirm(button.dataset.confirm)) event.preventDefault();\n  });\n});\n\n// After "Restart Pi": wait, then poll until the camera answers again.\n(() => {\n  const waiting = document.getElementById("wait-for-restart");\n  if (!waiting) return;\n  const poll = () => {\n    fetch("/login", { cache: "no-store" })\n      .then((response) => { if (response.ok) window.location.href = "/"; else setTimeout(poll, 5000); })\n      .catch(() => setTimeout(poll, 5000));\n  };\n  setTimeout(poll, Number(waiting.dataset.after || 30) * 1000);\n})();\n'),
    ('web/static/style.css',
     '.save-bar { display: flex; flex-wrap: wrap; align-items: flex-end; gap: 1rem; position: sticky; bottom: 0; }\n.save-bar .field { border: 0; flex: 1; min-width: 240px; }\n',
     '.save-bar { display: flex; flex-wrap: wrap; align-items: flex-end; gap: 1rem; position: sticky; bottom: 0; }\n.save-bar .field { border: 0; flex: 1; min-width: 240px; }\n\n/* ---- Phase 13b: time zone, restart and shut down ---- */\n.system-grid { margin-top: 1rem; }\n.system-grid form { display: grid; gap: .6rem; }\n.system-grid form > .button { justify-self: start; }\n.button-row { display: flex; flex-wrap: wrap; gap: .6rem; }\n.button.danger { background: #4a1f24; border-color: var(--bad); }\n.button.danger:hover { border-color: #ff8a8a; }\n'),
    ('deploy/install.sh',
     'install -m 0644 "$APP/deploy/journald-surveillance.conf" /etc/systemd/journald.conf.d/50-surveillance.conf\ninstall -m 0755 "$APP/deploy/cctv-tool" /usr/local/sbin/cctv-tool\nsystemctl daemon-reload\nsystemctl try-restart systemd-journald.service || true\n',
     'install -m 0644 "$APP/deploy/journald-surveillance.conf" /etc/systemd/journald.conf.d/50-surveillance.conf\ninstall -m 0755 "$APP/deploy/cctv-tool" /usr/local/sbin/cctv-tool\nif [ -d /etc/polkit-1/rules.d ]; then\n    install -m 0644 "$APP/deploy/polkit-surveillance.rules" /etc/polkit-1/rules.d/50-surveillance.rules\n    echo "polkit rule installed (Restart / Shut down / time zone from the web page)"\nelse\n    echo "WARNING: polkit is not installed, so Restart / Shut down / time zone will not work from the web page."\n    echo "         Install it with: sudo apt install polkitd   then run this script again."\nfi\nsystemctl daemon-reload\nsystemctl try-restart systemd-journald.service || true\n'),
]

NEW_FILES = [
    ('app/system_control.py', 0o644,
     '"""Restart / shut down the Pi and set its time zone from the web interface (Phase 13b).\n\nThe browser can only choose one of these fixed actions; no command is ever built from what it\nsends. The time zone must be one of the system\'s own zone names, and the commands run without\na shell. The operating system then decides whether the caller may do it: the polkit rule in\ndeploy/polkit-surveillance.rules allows exactly these actions to surveillance-web.service and\nnothing else, so running the same code anywhere else is refused.\n"""\nfrom __future__ import annotations\n\nimport logging\nimport os\nimport re\nimport subprocess\nimport threading\nimport zoneinfo\n\nlog = logging.getLogger("System")\n\n# action -> (logind "Can..." method, systemctl verb)\nPOWER_ACTIONS = {"reboot": ("CanReboot", "reboot"), "poweroff": ("CanPowerOff", "poweroff")}\nZONE_REGIONS = ("Africa", "America", "Antarctica", "Arctic", "Asia", "Atlantic", "Australia", "Europe",\n                "Indian", "Pacific")\nZONE_PATTERN = re.compile(r"^[A-Za-z]+(/[A-Za-z0-9_+-]+){1,2}$")\nCOMMAND_TIMEOUT_S = 20\n\n\nclass SystemControlError(RuntimeError):\n    pass\n\n\ndef _run(args: list[str]) -> subprocess.CompletedProcess:\n    try:\n        return subprocess.run(args, capture_output=True, text=True, timeout=COMMAND_TIMEOUT_S, check=False,\n                              stdin=subprocess.DEVNULL)\n    except (OSError, subprocess.TimeoutExpired) as exc:\n        raise SystemControlError(f"{args[0]} failed: {exc}") from exc\n\n\ndef _message(result: subprocess.CompletedProcess) -> str:\n    text = (result.stderr or result.stdout or "").strip().splitlines()\n    return text[-1][:200] if text else f"exit code {result.returncode}"\n\n\ndef time_zones() -> list[str]:\n    zones = {zone for zone in zoneinfo.available_timezones()\n             if zone.split("/")[0] in ZONE_REGIONS and ZONE_PATTERN.match(zone)}\n    return ["UTC", *sorted(zones)]\n\n\ndef current_time_zone() -> str | None:\n    try:\n        result = _run(["timedatectl", "show", "-p", "Timezone", "--value"])\n        if result.returncode == 0 and result.stdout.strip():\n            return result.stdout.strip()\n    except SystemControlError:\n        pass\n    try:\n        target = os.readlink("/etc/localtime")\n    except OSError:\n        return None\n    return target.split("zoneinfo/", 1)[1] if "zoneinfo/" in target else None\n\n\ndef set_time_zone(zone: str, allowed: list[str]) -> None:\n    if zone not in allowed:\n        raise SystemControlError("Unknown time zone.")\n    result = _run(["timedatectl", "set-timezone", zone])\n    if result.returncode != 0:\n        raise SystemControlError(f"The system refused to change the time zone: {_message(result)}")\n\n\ndef check_power(action: str) -> None:\n    """Ask logind whether this process may do it now, so a refusal is shown on the page."""\n    method = POWER_ACTIONS[action][0]\n    result = _run(["busctl", "call", "org.freedesktop.login1", "/org/freedesktop/login1",\n                   "org.freedesktop.login1.Manager", method])\n    answer = result.stdout.strip().removeprefix("s ").strip(\'"\') if result.returncode == 0 else ""\n    if answer != "yes":\n        detail = answer or _message(result)\n        raise SystemControlError(f"The system does not allow this ({detail}). Is the polkit rule installed? "\n                                 "Run sudo bash ~/surveillance/deploy/install.sh on the Pi.")\n\n\ndef schedule_power(action: str, delay_s: float) -> None:\n    """Run it a moment later, so the confirmation page reaches the browser first."""\n    def run() -> None:\n        try:\n            result = _run(["systemctl", POWER_ACTIONS[action][1]])\n        except SystemControlError as exc:\n            log.error("%s failed: %s", action, exc)\n            return\n        if result.returncode != 0:\n            log.error("%s failed: %s", action, _message(result))\n\n    threading.Timer(delay_s, run).start()\n'),
    ('deploy/polkit-surveillance.rules', 0o644,
     '// Installed as /etc/polkit-1/rules.d/50-surveillance.rules by deploy/install.sh (Phase 13b).\n// Lets the camera\'s web interface restart or shut down the Pi and set its time zone, and nothing\n// else. Only processes of user "cctv" inside surveillance-web.service qualify: the recorder runs\n// as the same user and is still refused. The web page asks for the account password each time.\npolkit.addRule(function (action, subject) {\n    var allowed = [\n        "org.freedesktop.login1.reboot",\n        "org.freedesktop.login1.reboot-multiple-sessions",\n        "org.freedesktop.login1.reboot-ignore-inhibit",\n        "org.freedesktop.login1.power-off",\n        "org.freedesktop.login1.power-off-multiple-sessions",\n        "org.freedesktop.login1.power-off-ignore-inhibit",\n        "org.freedesktop.timedate1.set-timezone"\n    ];\n    if (subject.user === "cctv" && subject.system_unit === "surveillance-web.service" &&\n            allowed.indexOf(action.id) >= 0) {\n        return polkit.Result.YES;\n    }\n    return polkit.Result.NOT_HANDLED;\n});\n'),
    ('web/templates/power.html', 0o644,
     '{% extends "base.html" %}\n{% block scripts %}<script src="{{ url_for(\'static\', filename=\'settings.js\') }}" defer></script>{% endblock %}\n{% block content %}\n<section class="panel">\n  {% if action == "reboot" %}\n    <h2>Restarting the Pi</h2>\n    <p>The Pi restarts in a few seconds. Recording stops for about a minute, then everything starts by itself.</p>\n    <p id="wait-for-restart" data-after="30" class="help">This page reconnects by itself when the camera is back.</p>\n  {% else %}\n    <h2>Shutting down the Pi</h2>\n    <p>The Pi shuts down in a few seconds and <strong>stays off</strong>. Wait until its green light has stopped\n      flashing, then unplug the power. Plugging it back in starts it again.</p>\n  {% endif %}\n</section>\n{% endblock %}\n'),
]

texts = {}
for rel, old, new in EDITS:
    text = texts.setdefault(rel, Path(rel).read_text())
    if text.count(old) != 1:
        sys.exit(f"ABORTED, nothing changed: {rel} does not match the expected 0.13.1 code "
                 f"(found {text.count(old)} matches for:\n{old})")
    texts[rel] = text.replace(old, new)
for rel, _mode, content in NEW_FILES:
    if Path(rel).exists() and Path(rel).read_text() != content:
        sys.exit(f"ABORTED, nothing changed: {rel} already exists with different content")
for rel, text in texts.items():
    shutil.copy2(rel, rel + ".bak")
    Path(rel).write_text(text)
    print(f"updated {rel}  (backup: {rel}.bak)")
for rel, mode, content in NEW_FILES:
    Path(rel).parent.mkdir(parents=True, exist_ok=True)
    Path(rel).write_text(content)
    os.chmod(rel, mode)
    print(f"created {rel}")
print("Update to 0.13.2 complete.")
PYEOF
python3 update_to_0_13_2.py
sha256sum app/__init__.py app/main.py app/web_settings.py app/system_control.py web/templates/settings.html web/templates/power.html web/static/settings.js web/static/style.css deploy/install.sh deploy/polkit-surveillance.rules
```

Expected checksums:

```
0b5d5ba0e3f7e13208aa526914efac646843d9c1147560e601f30ed5e374addd  app/__init__.py
248130a6b9a1966d701199edef6b0a05719b98a570be20492f24d8cc625f58b3  app/main.py
5a4e8f848ead733d0b971991ba44c10cab413b82476f380c77918a60efb921bb  app/web_settings.py
bf0ab9aaceb972a936e619b295f0efabebdb223c18cc01b0cc6bb2ea910d8373  app/system_control.py
2f319fb505310a186b81df0ef8efd7a29742847341ad42fa9b47058aa9f6cf27  web/templates/settings.html
45390f00fd175dd96b3945525952e49c32b616d544300982597129c758b9d13f  web/templates/power.html
7927b422d269c472ee4c985452259279141c5f23833b6c1b40a28247b33dee82  web/static/settings.js
10bab73a5a51356050575eaae19df59964b129c8dc7690cd83a7e85d62e0cec9  web/static/style.css
d395288187f59d6faa76fff9ff5958e35f6ce87e19e9892a8d7596e364486514  deploy/install.sh
d237237ded8de53942bd8d3f26a717499f2ba625dadd0aec6920c35372a41f16  deploy/polkit-surveillance.rules
```

## Step 2 — Deploy

```bash
sudo bash ~/surveillance/deploy/install.sh
```

Expected: the usual output, plus the line `polkit rule installed (Restart / Shut down / time zone from the web page)`, then three services each `active, 0 restart(s)`. If it prints the polkit WARNING instead, run `sudo apt install polkitd` and run the installer again.

## Step 3 — Check that nothing else gets the permission

These run a command as `cctv`, but outside the web service:

```bash
sudo systemd-run --quiet --wait --pipe --uid=cctv --gid=cctv timedatectl set-timezone UTC
sudo systemd-run --quiet --wait --pipe --uid=cctv --gid=cctv busctl call org.freedesktop.login1 /org/freedesktop/login1 org.freedesktop.login1.Manager CanReboot
timedatectl show -p Timezone --value
```

Expected: `Failed to set time zone: Interactive authentication required.`, then `s "challenge"`, then your unchanged zone (probably `Asia/Kuala_Lumpur`).

## Step 4 — Use it from the website

Open `https://cam01.tail1c1671.ts.net/settings` and scroll to the bottom.

1. **Time zone:** the dropdown should show your current zone. Pick another zone with the same offset (for example `Asia/Singapore`), click **Set time zone**, and wait for the page to reload. Then run `timedatectl show -p Timezone --value` on the Pi; it should print the new zone. Change it back afterwards.
2. **Wrong password:** type a wrong password into the Restart box and click **Restart Pi**, then OK in the dialog. Expected: `That password is wrong.`, and nothing restarts.
3. **Real restart:** enter the right password, click **Restart Pi**, then OK. You should see "Restarting the Pi", and after about a minute the page should return to the Dashboard by itself, showing ONLINE.
4. **Audit log:** open **Account** (your user name in the top bar). The latest entries should include `timezone changed`, `reauth failed` and `reboot`.

If a step shows "The system does not allow this (challenge)", your polkit build isn't seeing the web service's unit. Send me that message and the output of `dpkg -l polkitd dbus | tail -2`.

Only test **Shut down Pi** when you can reach the Pi, because it stays off until you unplug and replug its power.

Please send me the checksums and the results of Steps 2 to 4. Next is Phase 14 (hardening):
- SSH with keys only and no root login.
- SSH reachable only over the tailnet, enforced by a firewall.
- Automatic security updates.
- A final audit of ports and file permissions.
