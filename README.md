The audit caught something real. Your SSH hardening isn't in effect in the config files any more, even though it worked when you tested it. The firewall still blocks SSH from your home network, so right now passwords are refused from home but could be accepted over the tailnet.

Your other requests are in update 0.15.0 below. Phase 15 is skipped, and the full guides for GitHub come in my next message.

**The SSH finding.** `harden.sh` printed the hardened values, and your password test was refused. Ten minutes later, `sshd -T` shows exactly Raspberry Pi OS's default values. So the settings file was either deleted, or the `Include` line that makes SSH read the `sshd_config.d` folder was removed from `/etc/ssh/sshd_config`. The SSH server still has the hardened settings in memory, but the next restart or reboot would load the defaults.

- **In 0.15.0:** `harden.sh ssh` adds the `Include` line if it's missing and refuses to finish unless passwords really end up off. The audit now shows whether the file exists and is being read.
- **Before updating, please run this** so we know what changed it:

```bash
ls -l --time-style=+%H:%M /etc/ssh/sshd_config.d/ /etc/ssh/sshd_config
grep -n "^Include" /etc/ssh/sshd_config
sudo sshd -T | grep -E "^(passwordauthentication|permitrootlogin|allowusers) "
```

**The home-network test.** `Test-NetConnection` is Windows PowerShell only; your "command not found" wording suggests Git Bash or WSL. Use this there instead, with the Pi's home address from `hostname -I` on the Pi:

```bash
ssh -o ConnectTimeout=5 ysak@192.168.0.50
```

Expected: `Connection timed out` after 5 seconds.

**Skipping Phase 15** means some things were never tested on your Pi:
- a 72-hour run;
- a truly full SD card;
- unplugging the camera while it records;
- losing Wi-Fi.

I did test crash, hang, power-cut and reboot recovery in the test container, and a crash, a hang and a reboot on your Pi. If one of the untested cases ever happens, the dashboard shows it, and the recorder retries by itself.

## What's in 0.15.0

**Viewer accounts.** Every account is now either an *admin* or a *viewer*.

| | Admin | Viewer |
|---|---|---|
| Dashboard, Live View, Recordings (play and download), Motion Events, Storage, System Status | ✓ | ✓ |
| Settings, time zone, Restart / Shut down | ✓ | refused, with a "Not allowed" page |
| Change own password | ✓ | ✗ (the admin sets it on the Pi) |
| Security log | ✓ | ✗ |

- **Admin-only, even for hand-crafted requests:** the refusal applies to direct requests, not just hidden buttons. Posting straight to `/settings`, `/settings/timezone`, `/system/power` and `/account/password` with a valid CSRF token still gets 403.
- **Managed on the Pi only:** accounts are managed with `cctv-tool` (see Step 4). The last admin can't be demoted or deleted.
- **Existing accounts:** your current account becomes admin automatically.

**Moving to a new Pi:**
- `deploy/install-packages.sh` installs everything from apt on a fresh Pi. I checked that every package name exists in the Debian and Raspberry Pi Trixie archives.
- `deploy/backup.sh` backs up settings, accounts and the recording database, with `--with-recordings` for the videos too. It uses SQLite's own backup function, so the copy is consistent while recording continues.
- `deploy/restore.sh` puts a backup onto a new Pi.
- `.gitignore` keeps recordings, databases, logs, your settings and update leftovers out of GitHub.
- `README.md` is the front page of the repository.

I tested it on the Debian 13 test system:
- **Roles:** 22 browser checks passed for a viewer and an admin, plus the account commands and the last-admin guard.
- **Backup and restore:** a full new-Pi move worked. I backed up 292 recordings (2 GB), wiped all settings and data, restored, and re-ran the installer. Accounts, recordings, the index and the web login all came back.
- **Git:** a trial `git add` included no private or leftover files.

## Step 1 — Apply the update

```bash
cd ~/surveillance
cat > update_to_0_15_0.py <<'PYEOF'
#!/usr/bin/env python3
"""Update the surveillance project from 0.14.0 to 0.15.0 (run from ~/surveillance)."""
import os, shutil, sys
from pathlib import Path

EDITS = [
    ('app/__init__.py',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.14.0"\n',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.15.0"\n'),
    ('app/auth.py',
     'MAX_PASSWORD_LENGTH = 256\nUSERNAME_RE = re.compile(r"^[a-z][a-z0-9_.-]{2,31}$")\nTOKEN_BYTES = 32\nMAX_SESSIONS_PER_USER = 10\n',
     'MAX_PASSWORD_LENGTH = 256\nUSERNAME_RE = re.compile(r"^[a-z][a-z0-9_.-]{2,31}$")\n# admin: everything. viewer: watch live, browse and download recordings; no settings, no restart\n# or shutdown, cannot change any password (the administrator sets it on the Pi).\nROLES = ("admin", "viewer")\nTOKEN_BYTES = 32\nMAX_SESSIONS_PER_USER = 10\n'),
    ('app/auth.py',
     '    password_hash       TEXT NOT NULL,\n    created_at          INTEGER NOT NULL,\n    password_changed_at INTEGER NOT NULL\n);\nCREATE TABLE IF NOT EXISTS sessions (\n',
     "    password_hash       TEXT NOT NULL,\n    created_at          INTEGER NOT NULL,\n    password_changed_at INTEGER NOT NULL,\n    role                TEXT NOT NULL DEFAULT 'admin'\n);\nCREATE TABLE IF NOT EXISTS sessions (\n"),
    ('app/auth.py',
     '            with closing(self._connect()) as conn:\n                conn.executescript(SCHEMA)\n        finally:\n            os.umask(old_umask)\n',
     '            with closing(self._connect()) as conn:\n                conn.executescript(SCHEMA)\n                columns = {row["name"] for row in conn.execute("PRAGMA table_info(users)")}\n                if "role" not in columns:  # accounts made before roles existed are administrators\n                    try:\n                        conn.execute("ALTER TABLE users ADD COLUMN role TEXT NOT NULL DEFAULT \'admin\'")\n                    except sqlite3.OperationalError as exc:\n                        if "duplicate column" not in str(exc):\n                            raise\n        finally:\n            os.umask(old_umask)\n'),
    ('app/auth.py',
     '    def list_users(self) -> list[sqlite3.Row]:\n        with closing(self._connect()) as conn:\n            return conn.execute("SELECT u.id, u.username, u.created_at, u.password_changed_at,"\n                                " COALESCE(l.failed, 0) AS failed_logins, COALESCE(l.locked_until, 0) AS locked_until,"\n                                " (SELECT COUNT(*) FROM sessions s WHERE s.user_id = u.id) AS sessions"\n',
     '    def list_users(self) -> list[sqlite3.Row]:\n        with closing(self._connect()) as conn:\n            return conn.execute("SELECT u.id, u.username, u.role, u.created_at, u.password_changed_at,"\n                                " COALESCE(l.failed, 0) AS failed_logins, COALESCE(l.locked_until, 0) AS locked_until,"\n                                " (SELECT COUNT(*) FROM sessions s WHERE s.user_id = u.id) AS sessions"\n'),
    ('app/auth.py',
     '                                " ORDER BY u.username").fetchall()\n\n    def create_user(self, username: str, password: str) -> None:\n        username = normalize_username(username)\n        if not USERNAME_RE.match(username):\n            raise AuthError("Usernames are 3-32 characters: lowercase letters, digits, and . _ - (starting with a letter).")\n        problem = password_problem(password, username)\n        if problem:\n',
     '                                " ORDER BY u.username").fetchall()\n\n    def create_user(self, username: str, password: str, role: str = "admin") -> None:\n        username = normalize_username(username)\n        if not USERNAME_RE.match(username):\n            raise AuthError("Usernames are 3-32 characters: lowercase letters, digits, and . _ - (starting with a letter).")\n        if role not in ROLES:\n            raise AuthError(f"The role must be one of: {\', \'.join(ROLES)}.")\n        problem = password_problem(password, username)\n        if problem:\n'),
    ('app/auth.py',
     '        with closing(self._connect()) as conn:\n            try:\n                conn.execute("INSERT INTO users (username, password_hash, created_at, password_changed_at)"\n                             " VALUES (?, ?, ?, ?)", (username, self._hasher.hash(password), now, now))\n            except sqlite3.IntegrityError:\n                raise AuthError(f"User {username} already exists.") from None\n            self.audit(conn, username, "user_created")\n\n    def set_password(self, username: str, password: str, actor: str = "cli") -> None:\n',
     '        with closing(self._connect()) as conn:\n            try:\n                conn.execute("INSERT INTO users (username, password_hash, created_at, password_changed_at, role)"\n                             " VALUES (?, ?, ?, ?, ?)", (username, self._hasher.hash(password), now, now, role))\n            except sqlite3.IntegrityError:\n                raise AuthError(f"User {username} already exists.") from None\n            self.audit(conn, username, "user_created", f"role {role}")\n\n    def _check_not_last_admin(self, conn: sqlite3.Connection, username: str) -> None:\n        user = conn.execute("SELECT role FROM users WHERE username = ?", (username,)).fetchone()\n        if user is None:\n            raise AuthError(f"No user called {username}.")\n        admins = conn.execute("SELECT COUNT(*) FROM users WHERE role = \'admin\'").fetchone()[0]\n        if user["role"] == "admin" and admins <= 1:\n            raise AuthError(f"{username} is the only administrator; make another user admin first.")\n\n    def set_role(self, username: str, role: str) -> None:\n        username = normalize_username(username)\n        if role not in ROLES:\n            raise AuthError(f"The role must be one of: {\', \'.join(ROLES)}.")\n        with closing(self._connect()) as conn:\n            if role != "admin":\n                self._check_not_last_admin(conn, username)\n            if not conn.execute("UPDATE users SET role = ? WHERE username = ?", (role, username)).rowcount:\n                raise AuthError(f"No user called {username}.")\n            self.audit(conn, username, "role_changed", f"now {role}, by cli")\n\n    def delete_user(self, username: str) -> None:\n        username = normalize_username(username)\n        with closing(self._connect()) as conn:\n            self._check_not_last_admin(conn, username)\n            conn.execute("DELETE FROM users WHERE username = ?", (username,))\n            conn.execute("DELETE FROM lockouts WHERE username = ?", (username,))\n            self.audit(conn, username, "user_deleted", "by cli; all sessions ended")\n\n    def set_password(self, username: str, password: str, actor: str = "cli") -> None:\n'),
    ('app/auth.py',
     '        now = int(time.time())\n        with closing(self._connect()) as conn:\n            row = conn.execute("SELECT s.*, u.username FROM sessions s JOIN users u ON u.id = s.user_id"\n                               " WHERE s.token_hash = ?", (hash_token(token),)).fetchone()\n            if row is None:\n',
     '        now = int(time.time())\n        with closing(self._connect()) as conn:\n            row = conn.execute("SELECT s.*, u.username, u.role FROM sessions s JOIN users u ON u.id = s.user_id"\n                               " WHERE s.token_hash = ?", (hash_token(token),)).fetchone()\n            if row is None:\n'),
    ('app/auth.py',
     '        with closing(self._connect()) as conn:\n            user = conn.execute("SELECT * FROM users WHERE id = ?", (session["user_id"],)).fetchone()\n        if not self._verify(user["password_hash"], current[:MAX_PASSWORD_LENGTH]):\n            self.record(user["username"], "password_change_failed", "wrong current password")\n',
     '        with closing(self._connect()) as conn:\n            user = conn.execute("SELECT * FROM users WHERE id = ?", (session["user_id"],)).fetchone()\n        if user["role"] != "admin":\n            raise AuthError("Only the administrator can change passwords.")\n        if not self._verify(user["password_hash"], current[:MAX_PASSWORD_LENGTH]):\n            self.record(user["username"], "password_change_failed", "wrong current password")\n'),
    ('app/web_auth.py',
     'PUBLIC_ENDPOINTS = {"auth.login", "auth.login_post", "static"}\nRESOURCE_ENDPOINTS = {"api_status", "live.stream", "live.snapshot", "rec.video", "rec.download"}\nSAFE_METHODS = {"GET", "HEAD", "OPTIONS"}\nMESSAGES = {\n',
     'PUBLIC_ENDPOINTS = {"auth.login", "auth.login_post", "static"}\nRESOURCE_ENDPOINTS = {"api_status", "live.stream", "live.snapshot", "rec.video", "rec.download"}\n# Viewers may watch and download, but never reach these.\nADMIN_ENDPOINTS = {"cfg.settings_page", "cfg.save", "cfg.save_time_zone", "cfg.power", "auth.change_password"}\nSAFE_METHODS = {"GET", "HEAD", "OPTIONS"}\nMESSAGES = {\n'),
    ('app/web_auth.py',
     '                abort(401)\n            return redirect(url_for("auth.login", next=request.full_path.rstrip("?"), msg="expired"))\n        if request.method not in SAFE_METHODS:\n            sent = request.form.get("csrf_token") or request.headers.get("X-CSRF-Token", "")\n',
     '                abort(401)\n            return redirect(url_for("auth.login", next=request.full_path.rstrip("?"), msg="expired"))\n        if request.endpoint in ADMIN_ENDPOINTS and g.session.get("role") != "admin":\n            return render_template("placeholder.html", title="Not allowed", page="", phase=None,\n                                   text="Your account can watch the camera and its recordings, but only the "\n                                        "administrator can change settings, restart the camera or change "\n                                        "passwords."), 403\n        if request.method not in SAFE_METHODS:\n            sent = request.form.get("csrf_token") or request.headers.get("X-CSRF-Token", "")\n'),
    ('app/web_auth.py',
     '        session = getattr(g, "session", None)\n        return {"current_user": session["username"] if session else None,\n                "csrf_token": session["csrf_token"] if session else ""}\n\n',
     '        session = getattr(g, "session", None)\n        return {"current_user": session["username"] if session else None,\n                "is_admin": bool(session and session.get("role") == "admin"),\n                "csrf_token": session["csrf_token"] if session else ""}\n\n'),
    ('app/web_auth.py',
     '        audit = [{"at": datetime.fromtimestamp(a["at"]), "username": a["username"] or "",\n                  "action": a["action"].replace("_", " "), "detail": a["detail"] or ""}\n                 for a in store.recent_audit(25)]\n        return render_template("account.html", title="Account", page="auth.account", sessions=sessions,\n                               audit=audit, error=None, message=MESSAGES.get(request.args.get("msg", "")))\n',
     '        audit = [{"at": datetime.fromtimestamp(a["at"]), "username": a["username"] or "",\n                  "action": a["action"].replace("_", " "), "detail": a["detail"] or ""}\n                 for a in store.recent_audit(25)] if g.session.get("role") == "admin" else []\n        return render_template("account.html", title="Account", page="auth.account", sessions=sessions,\n                               audit=audit, error=None, message=MESSAGES.get(request.args.get("msg", "")))\n'),
    ('tools/manage_users.py',
     '"""Manage web accounts (run on the Pi over SSH; there is deliberately no web sign-up page).\n\n    python3 ~/surveillance/tools/manage_users.py create admin     # asks for the password twice\n    python3 ~/surveillance/tools/manage_users.py passwd admin     # set a new password, ends all sessions\n    python3 ~/surveillance/tools/manage_users.py list\n    python3 ~/surveillance/tools/manage_users.py unlock [admin]   # clear failed-login lockouts\n    python3 ~/surveillance/tools/manage_users.py logout admin     # end every session of the user\n    python3 ~/surveillance/tools/manage_users.py log              # recent security events\n"""\nfrom __future__ import annotations\n',
     '"""Manage web accounts (run on the Pi over SSH; there is deliberately no web sign-up page).\n\n    sudo cctv-tool manage_users create admin                # asks for the password twice\n    sudo cctv-tool manage_users create mum --role viewer    # can watch and download, nothing else\n    sudo cctv-tool manage_users role mum admin              # change a role (admin / viewer)\n    sudo cctv-tool manage_users passwd mum                  # set a new password, ends all sessions\n    sudo cctv-tool manage_users delete mum                  # remove the account and its sessions\n    sudo cctv-tool manage_users list\n    sudo cctv-tool manage_users unlock [admin]              # clear failed-login lockouts\n    sudo cctv-tool manage_users logout mum                  # end every session of the user\n    sudo cctv-tool manage_users log                         # recent security events\n"""\nfrom __future__ import annotations\n'),
    ('tools/manage_users.py',
     'sys.path.insert(0, str(Path(__file__).resolve().parent.parent))\n\nfrom app.auth import AUTH_DB_NAME, AuthError, AuthStore  # noqa: E402\nfrom app.config import DEFAULT_CONFIG_PATH, ConfigError, load_settings  # noqa: E402\n\n',
     'sys.path.insert(0, str(Path(__file__).resolve().parent.parent))\n\nfrom app.auth import AUTH_DB_NAME, ROLES, AuthError, AuthStore  # noqa: E402\nfrom app.config import DEFAULT_CONFIG_PATH, ConfigError, load_settings  # noqa: E402\n\n'),
    ('tools/manage_users.py',
     '    parser.add_argument("--config", type=Path, default=DEFAULT_CONFIG_PATH)\n    sub = parser.add_subparsers(dest="command", required=True)\n    for name in ("create", "passwd", "logout"):\n        sub.add_parser(name).add_argument("username")\n    sub.add_parser("unlock").add_argument("username", nargs="?")\n',
     '    parser.add_argument("--config", type=Path, default=DEFAULT_CONFIG_PATH)\n    sub = parser.add_subparsers(dest="command", required=True)\n    create = sub.add_parser("create")\n    create.add_argument("username")\n    create.add_argument("--role", choices=ROLES, default="admin",\n                        help="viewer: watch and download only; admin: everything (default)")\n    role = sub.add_parser("role")\n    role.add_argument("username")\n    role.add_argument("role", choices=ROLES)\n    for name in ("passwd", "logout", "delete"):\n        sub.add_parser(name).add_argument("username")\n    sub.add_parser("unlock").add_argument("username", nargs="?")\n'),
    ('tools/manage_users.py',
     '    try:\n        if args.command == "create":\n            store.create_user(args.username, ask_password(args.username.lower()))\n            print(f"User {args.username.lower()} created. Log in at http://localhost:8080/login")\n        elif args.command == "passwd":\n            store.set_password(args.username, ask_password(args.username.lower()))\n',
     '    try:\n        if args.command == "create":\n            store.create_user(args.username, ask_password(args.username.lower()), args.role)\n            print(f"User {args.username.lower()} created ({args.role}).")\n        elif args.command == "role":\n            store.set_role(args.username, args.role)\n            print(f"{args.username.lower()} is now {args.role} (applies to open sessions immediately).")\n        elif args.command == "delete":\n            if input(f"Delete {args.username.lower()} and end their sessions? Type yes: ").strip() != "yes":\n                print("Nothing deleted.")\n                return 1\n            store.delete_user(args.username)\n            print(f"{args.username.lower()} deleted.")\n        elif args.command == "passwd":\n            store.set_password(args.username, ask_password(args.username.lower()))\n'),
    ('tools/manage_users.py',
     '            users = store.list_users()\n            if not users:\n                print("No users yet. Create one with: python3 tools/manage_users.py create admin")\n            for u in users:\n                locked = " LOCKED until " + when(u["locked_until"]) if u["locked_until"] > datetime.now().timestamp() else ""\n                print(f"{u[\'username\']:<20} created {when(u[\'created_at\'])}  password changed "\n                      f"{when(u[\'password_changed_at\'])}  sessions {u[\'sessions\']}  failed logins "\n                      f"{u[\'failed_logins\']}{locked}")\n',
     '            users = store.list_users()\n            if not users:\n                print("No users yet. Create one with: sudo cctv-tool manage_users create admin")\n            for u in users:\n                locked = " LOCKED until " + when(u["locked_until"]) if u["locked_until"] > datetime.now().timestamp() else ""\n                print(f"{u[\'username\']:<16} {u[\'role\']:<7} created {when(u[\'created_at\'])}  password changed "\n                      f"{when(u[\'password_changed_at\'])}  sessions {u[\'sessions\']}  failed logins "\n                      f"{u[\'failed_logins\']}{locked}")\n'),
    ('web/templates/base.html',
     '  <nav class="nav" aria-label="Main">\n    {% for endpoint, label in navigation %}\n      <a href="{{ url_for(endpoint) }}" {% if page == endpoint %}class="active" aria-current="page"{% endif %}>{{ label }}</a>\n    {% endfor %}\n    <span class="nav-spacer"></span>\n',
     '  <nav class="nav" aria-label="Main">\n    {% for endpoint, label in navigation %}\n      {% if is_admin or endpoint != \'cfg.settings_page\' %}\n      <a href="{{ url_for(endpoint) }}" {% if page == endpoint %}class="active" aria-current="page"{% endif %}>{{ label }}</a>\n      {% endif %}\n    {% endfor %}\n    <span class="nav-spacer"></span>\n'),
    ('web/templates/account.html',
     '{% if message %}<p class="notice">{{ message }}</p>{% endif %}\n\n<section class="panel">\n  <h2>Change password</h2>\n',
     '{% if message %}<p class="notice">{{ message }}</p>{% endif %}\n\n{% if is_admin %}\n<section class="panel">\n  <h2>Change password</h2>\n'),
    ('web/templates/account.html',
     '  <p class="muted">A long passphrase from a password manager is best. Changing it logs out every other session.</p>\n</section>\n\n{% if sessions %}\n',
     '  <p class="muted">A long passphrase from a password manager is best. Changing it logs out every other session.</p>\n</section>\n{% else %}\n<section class="panel">\n  <h2>Your account</h2>\n  <p>You are signed in as <strong>{{ current_user }}</strong> with a <strong>viewer</strong> account: you can watch\n    live, browse and download recordings. To change your password, ask the administrator.</p>\n</section>\n{% endif %}\n\n{% if sessions %}\n'),
    ('web/templates/login.html',
     '  {% if no_users %}\n  <p class="notice">No account exists yet. Create one on the Raspberry Pi (over SSH):<br>\n    <code>cd ~/surveillance &amp;&amp; python3 tools/manage_users.py create admin</code></p>\n  {% endif %}\n  <form method="post" action="{{ url_for(\'auth.login_post\') }}" class="stack">\n',
     '  {% if no_users %}\n  <p class="notice">No account exists yet. Create one on the Raspberry Pi (over SSH):<br>\n    <code>sudo cctv-tool manage_users create admin</code></p>\n  {% endif %}\n  <form method="post" action="{{ url_for(\'auth.login_post\') }}" class="stack">\n'),
    ('deploy/harden.sh',
     '\n    say "Installing $SSHD_DROPIN"\n    sed "s/^AllowUsers ADMIN_USER$/AllowUsers $ADMIN/" "$APP/deploy/sshd-hardening.conf" > "$SSHD_DROPIN.new"\n    chmod 0644 "$SSHD_DROPIN.new"\n',
     '\n    say "Installing $SSHD_DROPIN"\n    if ! grep -Eq \'^[[:space:]]*Include[[:space:]]+/etc/ssh/sshd_config\\.d/\\*\\.conf\' /etc/ssh/sshd_config; then\n        cp -p /etc/ssh/sshd_config /etc/ssh/sshd_config.before-surveillance\n        sed -i \'1i Include /etc/ssh/sshd_config.d/*.conf\' /etc/ssh/sshd_config\n        echo "added the missing \'Include /etc/ssh/sshd_config.d/*.conf\' line to /etc/ssh/sshd_config"\n        echo "(the previous file is /etc/ssh/sshd_config.before-surveillance)"\n    fi\n    sed "s/^AllowUsers ADMIN_USER$/AllowUsers $ADMIN/" "$APP/deploy/sshd-hardening.conf" > "$SSHD_DROPIN.new"\n    chmod 0644 "$SSHD_DROPIN.new"\n'),
    ('deploy/harden.sh',
     '        die "sshd rejected the new settings; they were removed again, nothing changed"\n    fi\n    systemctl reload ssh.service 2>/dev/null || systemctl reload sshd.service\n    echo "sshd reloaded (your current session stays open)"\n',
     '        die "sshd rejected the new settings; they were removed again, nothing changed"\n    fi\n    sshd -T | grep -qx \'passwordauthentication no\' || \\\n        die "sshd still allows passwords: another file overrides $SSHD_DROPIN. Send the output of: ls -l /etc/ssh/sshd_config.d/"\n    systemctl reload ssh.service 2>/dev/null || systemctl reload sshd.service\n    echo "sshd reloaded (your current session stays open)"\n'),
    ('tools/phase14_security_audit.py',
     '        key, _, value = line.partition(" ")\n        cfg.setdefault(key, value)\n    expected = {"passwordauthentication": "no", "kbdinteractiveauthentication": "no", "permitrootlogin": "no",\n                "pubkeyauthentication": "yes", "permitemptypasswords": "no", "x11forwarding": "no",\n',
     '        key, _, value = line.partition(" ")\n        cfg.setdefault(key, value)\n    dropin = Path("/etc/ssh/sshd_config.d/10-surveillance.conf")\n    report("PASS" if dropin.exists() else "FAIL", f"{dropin} " + ("present" if dropin.exists()\n           else "is missing: run sudo bash ~/surveillance/deploy/harden.sh ssh"))\n    try:\n        main_config = Path("/etc/ssh/sshd_config").read_text()\n    except OSError:\n        main_config = ""\n    included = re.search(r"(?m)^\\s*Include\\s+/etc/ssh/sshd_config\\.d/\\*\\.conf", main_config)\n    report("PASS" if included else "FAIL", "/etc/ssh/sshd_config reads sshd_config.d" if included\n           else "/etc/ssh/sshd_config has no \'Include /etc/ssh/sshd_config.d/*.conf\' line, so the drop-in is ignored")\n    expected = {"passwordauthentication": "no", "kbdinteractiveauthentication": "no", "permitrootlogin": "no",\n                "pubkeyauthentication": "yes", "permitemptypasswords": "no", "x11forwarding": "no",\n'),
]

NEW_FILES = [
    ('deploy/backup.sh', 0o755,
     '#!/bin/bash\n# Back up what is needed to rebuild this camera on another Pi: settings, web accounts (password\n# hashes, not passwords) and the recordings/motion database. The program itself is in git.\n#\n#   sudo bash ~/surveillance/deploy/backup.sh                     -> ~/cctv-backup-<date>.tar.gz\n#   sudo bash ~/surveillance/deploy/backup.sh --with-recordings   (also the videos: can be many GB)\n#\n# Then copy it to your laptop (and keep it private: it contains the account hashes):\n#   scp ysak@cam01:cctv-backup-*.tar.gz .\nset -euo pipefail\n\nCONF=/etc/surveillance/settings.json\nDB_DIR=/var/lib/surveillance/database\nREC_DIR=/var/lib/surveillance/recordings\n\ndie() { printf \'\\nERROR: %s\\n\' "$*" >&2; exit 1; }\n[ "$(id -u)" -eq 0 ] || die "run it with sudo: sudo bash $0"\n[ -f "$CONF" ] || die "$CONF not found: is the camera installed (deploy/install.sh)?"\nwith_recordings=0\n[ "${1:-}" = "--with-recordings" ] && with_recordings=1\n\nowner="${SUDO_USER:-root}"\nhome="$(getent passwd "$owner" | cut -d: -f6)"\nout="$home/cctv-backup-$(date +%Y%m%d-%H%M).tar.gz"\nwork="$(mktemp -d)"\nfinished=0\ntrap \'rm -rf "$work"; [ "$finished" = 1 ] || rm -f "$out"\' EXIT\nmkdir -p "$work/etc" "$work/database"\n\ncp -p "$CONF" "$work/etc/settings.json"\nfor db in surveillance.db auth.db; do\n    [ -f "$DB_DIR/$db" ] || continue\n    # SQLite\'s own backup: a consistent copy even while the camera is writing.\n    python3 - "$DB_DIR/$db" "$work/database/$db" <<\'PY\'\nimport sqlite3, sys\nsrc = sqlite3.connect(f"file:{sys.argv[1]}?mode=ro", uri=True)\ndst = sqlite3.connect(sys.argv[2])\nwith dst:\n    src.backup(dst)\ndst.close(); src.close()\nPY\ndone\n{\n    echo "created=$(date -Iseconds)"\n    echo "host=$(hostname)"\n    echo "version=$(python3 -c \'import sys; sys.path.insert(0, "/opt/surveillance"); import app; print(app.__version__)\' 2>/dev/null || echo unknown)"\n    echo "recordings=$with_recordings"\n} > "$work/BACKUP-INFO"\n\numask 077\nif [ "$with_recordings" = 1 ]; then\n    need_kb="$(du -sk "$REC_DIR" | cut -f1)"\n    free_kb="$(df -Pk "$home" | awk \'NR==2 {print $4}\')"\n    [ "$free_kb" -gt $((need_kb + 1048576)) ] || \\\n        die "not enough free space in $home for the recordings ($((need_kb / 1024)) MB needed)"\n    # The recorder keeps writing while this runs: skip the unfinished segment, and accept tar\'s\n    # exit code 1 ("a directory changed while it was read").\n    tar -C "$work" -czf "$out" --warning=no-file-changed --exclude=\'*.partial\' BACKUP-INFO etc database \\\n        -C "$(dirname "$REC_DIR")" "$(basename "$REC_DIR")" || [ $? -eq 1 ]\nelse\n    tar -C "$work" -czf "$out" BACKUP-INFO etc database\nfi\nchown "$owner:" "$out"\nchmod 600 "$out"\nfinished=1\necho "Backup written: $out ($(du -h "$out" | cut -f1))"\ntar -tzf "$out" | grep -Ev \'/$\' | grep -v \'^recordings/\' | sed \'s/^/  /\'\n[ "$with_recordings" = 1 ] && echo "  + $(tar -tzf "$out" | grep -c \'^recordings/.*\\.mp4$\') recording(s)"\necho "Copy it to your laptop:  scp $owner@cam01:$(basename "$out") ."\n'),
    ('deploy/restore.sh', 0o755,
     '#!/bin/bash\n# Put a backup made by deploy/backup.sh onto this Pi (a new one, or a rebuilt SD card).\n# Run it after deploy/install-packages.sh and before (or instead of re-running) deploy/install.sh:\n#\n#   sudo bash ~/surveillance/deploy/restore.sh ~/cctv-backup-20260924-2300.tar.gz\n#   sudo bash ~/surveillance/deploy/install.sh\n#\n# Anything already on this Pi is moved aside (*.before-restore-<date>), never deleted.\nset -euo pipefail\n\nCONF_DIR=/etc/surveillance\nDB_DIR=/var/lib/surveillance/database\nREC_DIR=/var/lib/surveillance/recordings\n\ndie() { printf \'\\nERROR: %s\\n\' "$*" >&2; exit 1; }\n[ "$(id -u)" -eq 0 ] || die "run it with sudo: sudo bash $0 BACKUP-FILE"\nbackup="${1:-}"\n[ -f "$backup" ] || die "usage: sudo bash $0 ~/cctv-backup-<date>.tar.gz"\n\nwork="$(mktemp -d)"\ntrap \'rm -rf "$work"\' EXIT\ntar -C "$work" -xzf "$backup"\n[ -f "$work/etc/settings.json" ] && [ -f "$work/BACKUP-INFO" ] || die "$backup is not a camera backup"\necho "Backup:"; sed \'s/^/  /\' "$work/BACKUP-INFO"\n\nsystemctl stop surveillance-web.service surveillance-recorder.service 2>/dev/null || true\nstamp="before-restore-$(date +%Y%m%d-%H%M%S)"\ninstall -d -m 0750 "$CONF_DIR" "$DB_DIR" "$REC_DIR"\n\nif [ -f "$CONF_DIR/settings.json" ]; then mv "$CONF_DIR/settings.json" "$CONF_DIR/settings.json.$stamp"; fi\ninstall -m 0640 "$work/etc/settings.json" "$CONF_DIR/settings.json"\nfor db in surveillance.db auth.db; do\n    [ -f "$work/database/$db" ] || continue\n    for old in "$DB_DIR/$db" "$DB_DIR/$db-wal" "$DB_DIR/$db-shm"; do\n        if [ -e "$old" ]; then mv "$old" "$old.$stamp"; fi\n    done\n    install -m 0640 "$work/database/$db" "$DB_DIR/$db"\ndone\nif [ -d "$work/recordings" ]; then\n    cp -a -n "$work/recordings/." "$REC_DIR/"\n    echo "recordings restored: $(find "$work/recordings" -name \'*.mp4\' | wc -l)"\nfi\nif id cctv >/dev/null 2>&1; then\n    chown -R cctv:cctv "$CONF_DIR" /var/lib/surveillance\nfi\nchmod -R u=rwX,g=rX,o= "$CONF_DIR" /var/lib/surveillance\n\nhostname="$(python3 -c \'import json; print(json.load(open("/etc/surveillance/settings.json"))["web"]["https_hostname"])\')"\ncat <<EOF\n\nRestored settings, accounts and the recording index.\nNext:  sudo bash ~/surveillance/deploy/install.sh\nThe web address in the backup is: ${hostname:-(none)}\nIf this Pi gets a different tailnet name, set it:  sudo cctv-tool set_setting web.https_hostname <new name>\nEOF\n'),
    ('deploy/install-packages.sh', 0o755,
     '#!/bin/bash\n# Fresh Raspberry Pi OS (Trixie, 64-bit): install every system package the camera needs.\n#   sudo bash ~/surveillance/deploy/install-packages.sh\n# Everything comes from the Debian and Raspberry Pi archives (no pip), so it is updated with apt.\nset -euo pipefail\n\n[ "$(id -u)" -eq 0 ] || { echo "run it with sudo: sudo bash $0" >&2; exit 1; }\n\nPACKAGES=(\n    python3-picamera2 python3-simplejpeg python3-numpy python3-opencv python3-av   # camera, video, motion\n    python3-flask python3-waitress python3-argon2                                   # web interface, logins\n    ffmpeg                                                                           # checking and repairing recordings\n    rpicam-apps                                                                      # rpicam-hello camera test\n    polkitd nftables git curl                                                        # restart button, firewall, tools\n)\n\napt-get update\n# --no-install-recommends keeps desktop libraries off a headless camera.\nDEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends "${PACKAGES[@]}"\necho\necho "Installed. Check the camera:  rpicam-hello --list-cameras"\n'),
    ('.gitignore', 0o644,
     '# Data and machine-specific files: never put these in git.\n/recordings/\n/database/\n/logs/\n/config/settings.json\n*.partial\ncctv-backup-*.tar.gz\n\n# Left over from updates and Python.\n*.bak\n/update_*.py\n__pycache__/\n*.pyc\n'),
    ('README.md', 0o644,
     '# Raspberry Pi security camera\n\nA self-hosted CCTV recorder for a Raspberry Pi 4 with a camera module: 24/7 recording, motion\nevents, live view and playback in the browser, reachable from your phone and laptop anywhere\nthrough a private Tailscale network. Nothing is exposed to the internet and no port forwarding\nis needed.\n\n## Features\n\n- **Recording around the clock** in 5-minute H.264 segments (hardware encoder), power-cut safe:\n  an interrupted segment is repaired on the next start.\n- **Storage limit**: oldest recordings are deleted first (never the one being written), with a\n  minimum amount of free space kept on the SD card.\n- **Motion detection** (OpenCV, on a small preview stream) with events, a timeline and jump-to-moment.\n- **Web interface**: dashboard, live view, recordings, motion events, storage, settings, system\n  status. Works on phones.\n- **Accounts**: Argon2id passwords, secure sessions, CSRF protection, lockouts, audit log;\n  *admin* and *viewer* roles (viewers can watch and download, nothing else).\n- **Runs as system services**: starts at boot, restarts itself after a crash or a hang\n  (systemd watchdog), sandboxed, least privilege.\n- **Hardened**: SSH with keys only and only through the tailnet, firewall, automatic security\n  updates, a security audit tool.\n\n## Guides\n\n| Guide | What it covers |\n|---|---|\n| [docs/INSTALL.md](docs/INSTALL.md) | Setting up a new Pi from scratch (or restoring a backup onto one) |\n| [docs/REMOTE_ACCESS.md](docs/REMOTE_ACCESS.md) | Tailscale: private HTTPS access from your phone and laptop |\n| [docs/SECURITY.md](docs/SECURITY.md) | Threat model, hardening, accounts and roles, lock-out recovery |\n| [docs/BACKUP.md](docs/BACKUP.md) | Backups, restoring, moving to another Pi |\n| [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) | Common problems and how to fix them |\n| [docs/PERFORMANCE.md](docs/PERFORMANCE.md) | Measured load, SD card sizing, tuning |\n| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | How the parts fit together |\n| [docs/DESIGN.md](docs/DESIGN.md) | The original design notes and the reasons behind the choices |\n\n## Everyday commands (on the Pi)\n\n```bash\nsystemctl status surveillance-recorder surveillance-web surveillance-health\njournalctl -u surveillance-recorder -u surveillance-web -f        # problems, live\nsudo tail -f /var/log/surveillance/surveillance.log               # full recorder log\nsudo cctv-tool manage_users list                                  # web accounts\nsudo cctv-tool manage_users create NAME --role viewer\nsudo cctv-tool set_setting --show                                 # all settings\nsudo cctv-tool phase4_check_segments --last 20                    # recordings healthy?\nsudo cctv-tool phase12_tailscale_check                            # remote access healthy?\nsudo python3 /opt/surveillance/tools/phase14_security_audit.py    # security audit\nsudo bash ~/surveillance/deploy/backup.sh                         # backup of settings and accounts\nsudo bash ~/surveillance/deploy/install.sh                        # deploy after changing files here\n```\n\n## Repository layout\n\n```\napp/        the program (recorder, storage manager, motion detector, database, web interface)\nweb/        page templates, styles and scripts\ntools/      checks and maintenance tools (run them with: sudo cctv-tool NAME)\ndeploy/     installer, systemd units, firewall, SSH hardening, backup and restore scripts\ndocs/       the guides above\nconfig/     development settings (not used once installed; the live file is /etc/surveillance)\n```\n\nInstalled layout: program in `/opt/surveillance` (read-only), settings in `/etc/surveillance`,\nrecordings and database in `/var/lib/surveillance`, logs in `/var/log/surveillance`.\n\n## Requirements\n\nRaspberry Pi 4 (2 GB or more), a Raspberry Pi camera module (tested with an OV5647),\nRaspberry Pi OS Trixie 64-bit, a 32 GB or larger SD card (64 GB recommended), a Tailscale\naccount. All software comes from the Debian and Raspberry Pi package archives; see\n[deploy/install-packages.sh](deploy/install-packages.sh).\n'),
]

texts = {}
for rel, old, new in EDITS:
    text = texts.setdefault(rel, Path(rel).read_text())
    if text.count(old) != 1:
        sys.exit(f"ABORTED, nothing changed: {rel} does not match the expected 0.14.0 code "
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
print("Update to 0.15.0 complete.")
PYEOF
python3 update_to_0_15_0.py
sha256sum app/__init__.py app/auth.py app/web_auth.py tools/manage_users.py web/templates/base.html web/templates/account.html web/templates/login.html deploy/harden.sh tools/phase14_security_audit.py deploy/backup.sh deploy/restore.sh deploy/install-packages.sh .gitignore README.md
sudo bash ~/surveillance/deploy/install.sh
```

Expected checksums:

```
ff87c9ecdeae5486e1acbf7d53787496b03812b942f15c89d5c98e672bafaf5c  app/__init__.py
c12d9a92add9f5ff3e28d1dec3c0272983b8439198ff260b753e2d33648e08bc  app/auth.py
fbff1bf9b27783de06f83880dcabc82d072974717178b4de833729a774dc78f3  app/web_auth.py
99d1b5c9c7ecf46faea70485b26a3c6ed0815534de7991a17591c577d2a099b1  tools/manage_users.py
b296d00d4af8d85b450cdeae3f70b97e466b083d1f45d205d30d979661d4e2b7  web/templates/base.html
c2fb6f7ac6bdff3df0b00bb471f95817f91d50f0ef3840751510ef866a704fda  web/templates/account.html
38cbe040f55510ab3177b84964d0b1e28074b0c1991e331e66a99aaa5418856d  web/templates/login.html
ec804ba2f21acfa50729efc719f0b6fe135e5916021bb7d3cb7ff14e99c05fc0  deploy/harden.sh
5f47263ef4da38d48cc5f3fc349f4c814a658ca7ecfe9be4a98a0b654ba6c7e9  tools/phase14_security_audit.py
d48adf64f7180489e57a29e8ede08a34200310defefdf25ce73d5ee41cdb1405  deploy/backup.sh
e4f7a060a6ac1713338589b2560696ec856be53271e36e35cc332782473fae1f  deploy/restore.sh
014a080d5512348103b1be86f4487b93f3de1cc5fe63fe77f1a7628d5ab50dd0  deploy/install-packages.sh
0f86a1658556b0bf6fc9ce7df499a89bac963557bde1dda1eb04c50851bad410  .gitignore
78238838d830b1fa890daf9eaf4716e6bb06e2b694c307cdc7bfdc892ead2041  README.md
```

The installer should end with `version 0.15.0` and three services each `active, 0 restart(s)`.

## Step 2 — Fix SSH and re-run the audit

```bash
sudo bash ~/surveillance/deploy/harden.sh ssh
sudo python3 /opt/surveillance/tools/phase14_security_audit.py
```

- **`harden.sh ssh`:** if the `Include` line was missing, it prints `added the missing 'Include ...' line`.
- **Audit:** it should end with 0 fail. The SSH lines should now show `passwordauthentication no`, `permitrootlogin no` and `allowusers ysak`.

Then, in a new laptop terminal, check that key login still works and passwords are still refused:

```bash
ssh ysak@cam01
ssh -o PubkeyAuthentication=no ysak@cam01
```

The first logs in with your key; the second prints `Permission denied (publickey)`.

## Step 3 — Add a viewer account

On the Pi:

```bash
sudo cctv-tool manage_users create mum --role viewer
sudo cctv-tool manage_users list
```

`create` asks for a password of at least 15 characters. `list` shows the role next to each name. Other commands:

| Command | What it does |
|---|---|
| `sudo cctv-tool manage_users passwd mum` | Set a new password (you do this, not the viewer) |
| `sudo cctv-tool manage_users role mum admin` | Promote to admin, or `viewer` to demote |
| `sudo cctv-tool manage_users delete mum` | Remove the account and end its sessions |

**Their phone also needs to be in your tailnet.** The best way is their own Tailscale account, with access to the camera page only:

1. In the admin console, go to **Users → Invite users** and invite them. They install Tailscale and sign in with their own account. Approve their device under **Machines**.
2. In **Access controls**, add a `groups` section and change `grants` to the version below. Your own devices keep HTTPS and SSH; family devices get the web page (443) only.

```json
  "groups": {
    "group:family": ["THEIR-LOGIN@example.com"]
  },
  "grants": [
    {"src": ["autogroup:member"], "dst": ["autogroup:self"], "ip": ["*"]},
    {"src": ["YOUR-LOGIN@example.com"], "dst": ["tag:camera"], "ip": ["tcp:443", "tcp:22"]},
    {"src": ["group:family"], "dst": ["tag:camera"], "ip": ["tcp:443"]}
  ],
```

3. Once they've joined, add this line inside `"tests"` so Tailscale checks the rule every time you save:

```json
    {"src": "THEIR-LOGIN@example.com", "accept": ["tag:camera:443"], "deny": ["tag:camera:22"]},
```

Then check it from their phone:
- Signing in shows the menu without **Settings**.
- Opening `/settings` shows "Not allowed".
- Live view, recordings and downloads work.

## Step 4 — Put the code on GitHub

Everything in `~/surveillance` goes into a **private** repository; your settings, recordings and database stay out.

1. On github.com, click **New repository**, give it a name (for example `cctv-pi`), choose **Private**, and don't add a README.
2. On the Pi, create a key that can reach only this one repository:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/github_cctv -N "" -C "cam01 -> github"
cat ~/.ssh/github_cctv.pub
```

3. In the repository on GitHub, open **Settings → Deploy keys → Add deploy key**, paste the line that `cat` printed, tick **Allow write access**, and save. Then on the Pi:

```bash
printf 'Host github.com\n    IdentityFile ~/.ssh/github_cctv\n    IdentitiesOnly yes\n' >> ~/.ssh/config
chmod 600 ~/.ssh/config
ssh -T git@github.com
```

The first connection asks you to confirm GitHub's fingerprint. It should be `SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU`; compare it with GitHub's "SSH key fingerprints" help page, then type `yes`. Expected: `Hi YOUR-NAME/cctv-pi! You've successfully authenticated, but GitHub does not provide shell access.`

4. Commit and push (replace the name, email and repository path):

```bash
cd ~/surveillance
git init -b main
git config user.name "Your Name"
git config user.email "you@example.com"
git add .
git ls-files | grep -E '^(recordings|database|logs)/|^config/settings\.json|\.bak$|^update_' || echo "nothing private included"
git commit -m "Raspberry Pi security camera 0.15.0"
git remote add origin git@github.com:YOUR-NAME/cctv-pi.git
git push -u origin main
```

The check before the commit must print `nothing private included`.

After any future update: `git add -A && git commit -m "Update to 0.x.y" && git push`.

## Step 5 — Make a first backup

```bash
sudo bash ~/surveillance/deploy/backup.sh
```

It writes `~/cctv-backup-<date>.tar.gz` (about 20 KB) and lists its contents. Copy it to your laptop and keep it private, because it contains the account password hashes:

```bash
scp ysak@cam01:cctv-backup-*.tar.gz .
```

Add `--with-recordings` if you want the videos too. That can be several GB.

## On a new Pi, in short

1. Flash Raspberry Pi OS Trixie 64-bit.
2. Install Tailscale and join your tailnet. Remove the old camera from **Machines** first, so the new one can take the name `cam01`.
3. Clone the repository and install the packages:

```bash
git clone git@github.com:YOUR-NAME/cctv-pi.git ~/surveillance
sudo bash ~/surveillance/deploy/install-packages.sh
```

4. Restore your backup, then install:

```bash
sudo bash ~/surveillance/deploy/restore.sh ~/cctv-backup-<date>.tar.gz
sudo bash ~/surveillance/deploy/install.sh
```

5. Set up Tailscale's HTTPS again (tag, key expiry, `tailscale serve`) and run the `harden.sh` steps.

The step-by-step guides are next: `docs/INSTALL.md`, `REMOTE_ACCESS.md`, `SECURITY.md`, `BACKUP.md`, `TROUBLESHOOTING.md`, `PERFORMANCE.md` and `ARCHITECTURE.md`. The README already links to them. I'm sending them as a separate paste because together with this update the block would be too large to paste reliably. They'll cover every step above in full, including a clean install without a backup.

Please send me:
1. The output of the three SSH diagnostic commands (before the update).
2. The checksums.
3. The Step 2 audit result.
4. Whether the viewer account behaves as described.
5. Whether `git push` worked.
