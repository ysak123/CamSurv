Phase 11 (login and security) is ready for you to test on the Pi. It passed all my tests here in Chrome and with scripted attacks.

## What Phase 11 adds

**Accounts are created only on the Pi, over SSH.** There's deliberately no sign-up page in the browser, which a stranger could otherwise reach first. `tools/manage_users.py` has `create`, `passwd`, `list`, `unlock`, `logout` and `log`.

**Passwords**
- **At least 15 characters,** and not the username or a very common password. There are no "must include a symbol" rules: a long passphrase is stronger.
- **Hashed with Argon2id** using the RFC 9106 settings (64 MiB, 3 passes, 4 lanes). Each check deliberately takes a few hundred milliseconds on the Pi, so guessing is slow and expensive.
- **At most two checks run at once,** so a flood of login attempts can't use up the Pi's memory.

**Protection against guessing**
- **Username lockout:** after 5 wrong passwords, that username is locked for 30 s, then 60 s, 2 min and so on, up to 15 min. **This applies to invented usernames too.**
- **Global limit:** more than 30 failures in 10 minutes pauses all logins for a while.
- The counters are stored in the database, so restarting doesn't reset them.
- If you ever lock yourself out, run `manage_users.py unlock`.

**Sessions**
- A 256-bit random token in a cookie that JavaScript can't read. It's only sent over secure connections, never to other websites, and only to this exact address.
- The database stores only a hash of the token, so a copy of the database contains no usable logins.
- You're logged out after 30 minutes idle, or 12 hours at most.
- Changing your password logs out every other session, and the Account page has a "Log out all other sessions" button.

**Every page, stream and file is protected.** Pages redirect to the login page. The API, live stream, snapshot, video and downloads answer `401`. Only the login page and the style/script files are public.

**Cross-site attacks (CSRF) are blocked.** Every form must come from the camera's own pages, and must carry a secret token unique to your session.

**Security log.** Logins, failures, lockouts, password changes and logouts are shown on the Account page and by `manage_users.py log`.

**Files:**
- `auth.db` is created readable only by your user (`-rw-------`).
- New settings in the `auth` section: `idle_timeout_minutes` (default 30), `session_max_hours` (12) and `lockout_threshold` (5).

## Two problems my tests caught and I fixed

1. **Every login would have failed.** The pages told browsers `Referrer-Policy: no-referrer`, and under that policy browsers label every form post as coming from origin `null`. My same-origin check would then have rejected all logins. It's now `same-origin`: the browser still never sends the camera's address to other websites.
2. **The lockout revealed which usernames exist.** Only real accounts got locked, so after 5 guesses an attacker would see "Too many attempts" for real usernames and "Wrong username or password" for invented ones. Every attempted username now gets the same treatment.

## What I tested here

**In Chrome:**
- The cookie was accepted with all its protections, even over plain `http://127.0.0.1` through the tunnel.
- JavaScript can't read the cookie.
- A same-site POST without the CSRF token got `403`.
- Changing the password needs the current one.
- A session idle for 60 minutes bounced to the login page.
- Logout ends the session on the server; the API then answers `401`.
- Live view, video playback and the dashboard snapshot all still work after logging in.

**With scripted attacks:**
- Every protected route answers `302` (pages) or `401` (everything else) without a login.
- A login from a foreign site got `403`.
- **A real and an invented username behave identically:** "Wrong username or password" at about 53 ms four times, then "Too many failed attempts" at about 10 ms.
- The correct password is also refused while the username is locked.

**Update script:** turns your 0.10.0 files into files identical to the tested ones, and aborts safely if run twice.

**Browser note:** Chrome, Edge and Firefox accept this kind of secure cookie on `http://localhost`. **Safari may not,** so please use one of the others through the tunnel for now. Once Tailscale gives the camera a real `https://` address in Phase 12, every browser works.

## Install

```bash
sudo apt install -y python3-argon2
python3 -c "import argon2; print('argon2 ok')"
```

## Update the files

**1. Update script.** It changes `__init__.py`, `config.py`, `web.py`, `base.html`, `app.js` and `style.css`, making `.bak` backups:

```bash
cd ~/surveillance
cat > update_to_0_11_0.py <<'EOF'
#!/usr/bin/env python3
"""Update the surveillance project from 0.10.0 to 0.11.0 (run from ~/surveillance)."""
import shutil, sys
from pathlib import Path

EDITS = [
    ('app/__init__.py',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.10.0"\n',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.11.0"\n'),
    ('app/config.py',
     '\n@dataclass(frozen=True)\nclass WebSettings:\n    host: str = "127.0.0.1"\n',
     '\n@dataclass(frozen=True)\nclass AuthSettings:\n    idle_timeout_minutes: int = 30\n    session_max_hours: int = 12\n    lockout_threshold: int = 5\n\n    def validate(self) -> None:\n        _check_range("auth.idle_timeout_minutes", self.idle_timeout_minutes, 5, 1440)\n        _check_range("auth.session_max_hours", self.session_max_hours, 1, 168)\n        _check_range("auth.lockout_threshold", self.lockout_threshold, 3, 20)\n\n\n@dataclass(frozen=True)\nclass WebSettings:\n    host: str = "127.0.0.1"\n'),
    ('app/config.py',
     '    motion: MotionSettings = field(default_factory=MotionSettings)\n    live: LiveSettings = field(default_factory=LiveSettings)\n    web: WebSettings = field(default_factory=WebSettings)\n    paths: PathSettings = field(default_factory=PathSettings)\n',
     '    motion: MotionSettings = field(default_factory=MotionSettings)\n    live: LiveSettings = field(default_factory=LiveSettings)\n    auth: AuthSettings = field(default_factory=AuthSettings)\n    web: WebSettings = field(default_factory=WebSettings)\n    paths: PathSettings = field(default_factory=PathSettings)\n'),
    ('app/web.py',
     'from app.status import read_status\nfrom app.system_info import SystemInfo\nfrom app.web_live import create_blueprint as create_live_blueprint\nfrom app.web_recordings import create_blueprint\n',
     'from app.status import read_status\nfrom app.system_info import SystemInfo\nfrom app.web_auth import install as install_auth\nfrom app.web_live import create_blueprint as create_live_blueprint\nfrom app.web_recordings import create_blueprint\n'),
    ('app/web.py',
     '    "X-Content-Type-Options": "nosniff",\n    "X-Frame-Options": "DENY",\n    "Referrer-Policy": "no-referrer",\n    "Permissions-Policy": "camera=(), microphone=(), geolocation=(), payment=(), usb=()",\n    "Cross-Origin-Opener-Policy": "same-origin",\n',
     '    "X-Content-Type-Options": "nosniff",\n    "X-Frame-Options": "DENY",\n    # same-origin (not no-referrer): with no-referrer browsers send "Origin: null" on form posts,\n    # which would defeat the same-origin check that protects every POST.\n    "Referrer-Policy": "same-origin",\n    "Permissions-Policy": "camera=(), microphone=(), geolocation=(), payment=(), usb=()",\n    "Cross-Origin-Opener-Policy": "same-origin",\n'),
    ('app/web.py',
     '            abort(400)\n\n    @app.after_request\n    def security_headers(response):\n',
     '            abort(400)\n\n    # Registered after the Host check, so it runs second: login, CSRF and same-origin checks.\n    install_auth(app, settings)\n\n    @app.after_request\n    def security_headers(response):\n'),
    ('web/templates/base.html',
     '  <meta charset="utf-8">\n  <meta name="viewport" content="width=device-width, initial-scale=1">\n  <meta name="referrer" content="no-referrer">\n  <link rel="icon" href="data:,">\n  <title>{{ title }} · {{ camera_name }}</title>\n',
     '  <meta charset="utf-8">\n  <meta name="viewport" content="width=device-width, initial-scale=1">\n  <meta name="referrer" content="same-origin">\n  <link rel="icon" href="data:,">\n  <title>{{ title }} · {{ camera_name }}</title>\n'),
    ('web/templates/base.html',
     '  {% block scripts %}{% endblock %}\n</head>\n<body data-refresh="{{ refresh_ms }}">\n  <header class="topbar">\n    <div class="brand">\n',
     '  {% block scripts %}{% endblock %}\n</head>\n<body data-refresh="{{ refresh_ms }}" data-auth="{{ \'1\' if current_user else \'0\' }}">\n  <header class="topbar">\n    <div class="brand">\n'),
    ('web/templates/base.html',
     '      </div>\n    </div>\n    <p class="clock" id="updated-at" aria-live="polite">connecting…</p>\n  </header>\n  <nav class="nav" aria-label="Main">\n    {% for endpoint, label in navigation %}\n      <a href="{{ url_for(endpoint) }}" {% if page == endpoint %}class="active" aria-current="page"{% endif %}>{{ label }}</a>\n    {% endfor %}\n  </nav>\n  <div class="banner" id="connection-banner" role="alert" hidden>Cannot reach the camera. Retrying…</div>\n  <main>\n',
     '      </div>\n    </div>\n    {% if current_user %}<p class="clock" id="updated-at" aria-live="polite">connecting…</p>{% endif %}\n  </header>\n  {% if current_user %}\n  <nav class="nav" aria-label="Main">\n    {% for endpoint, label in navigation %}\n      <a href="{{ url_for(endpoint) }}" {% if page == endpoint %}class="active" aria-current="page"{% endif %}>{{ label }}</a>\n    {% endfor %}\n    <span class="nav-spacer"></span>\n    <a href="{{ url_for(\'auth.account\') }}" {% if page == \'auth.account\' %}class="active" aria-current="page"{% endif %}>{{ current_user }}</a>\n    <form method="post" action="{{ url_for(\'auth.logout\') }}" class="nav-form">\n      <input type="hidden" name="csrf_token" value="{{ csrf_token }}">\n      <button type="submit" class="nav-button">Log out</button>\n    </form>\n  </nav>\n  {% endif %}\n  <div class="banner" id="connection-banner" role="alert" hidden>Cannot reach the camera. Retrying…</div>\n  <main>\n'),
    ('web/static/app.js',
     '// Only textContent is ever set, so nothing from the server can be interpreted as HTML.\n(() => {\n  const refreshMs = Number(document.body.dataset.refresh) || 5000;\n  let timer = null;\n',
     '// Only textContent is ever set, so nothing from the server can be interpreted as HTML.\n(() => {\n  if (document.body.dataset.auth !== "1") return;   // login page: nothing to poll\n  const refreshMs = Number(document.body.dataset.refresh) || 5000;\n  let timer = null;\n'),
    ('web/static/app.js',
     '    try {\n      const response = await fetch("/api/status", { cache: "no-store", credentials: "same-origin" });\n      if (!response.ok) throw new Error(`HTTP ${response.status}`);\n      render(await response.json());\n',
     '    try {\n      const response = await fetch("/api/status", { cache: "no-store", credentials: "same-origin" });\n      if (response.status === 401) {   // session expired or logged out elsewhere\n        window.location.href = `/login?msg=expired&next=${encodeURIComponent(location.pathname + location.search)}`;\n        return;\n      }\n      if (!response.ok) throw new Error(`HTTP ${response.status}`);\n      render(await response.json());\n'),
    ('web/static/style.css',
     '  letter-spacing: .05em;\n}\n',
     '  letter-spacing: .05em;\n}\n\n/* ---- Phase 11: login and account ---- */\n.nav { align-items: center; }\n.nav-spacer { flex: 1; }\n.nav-form { margin: 0; }\n.nav-button {\n  padding: .45rem .8rem;\n  border: 0;\n  border-radius: 8px;\n  background: none;\n  color: var(--muted);\n  font: inherit;\n  font-size: .9rem;\n  cursor: pointer;\n}\n.nav-button:hover { color: var(--text); background: var(--panel-2); }\n\n.login-panel { max-width: 420px; width: 100%; margin: 2rem auto; }\nform.stack { display: grid; gap: .9rem; }\nform.stack.narrow { max-width: 420px; }\nform.stack label { display: grid; gap: .3rem; color: var(--muted); font-size: .85rem; }\nform.stack input[type="text"], form.stack input[type="password"] {\n  padding: .6rem .7rem;\n  border: 1px solid var(--border);\n  border-radius: 8px;\n  background: var(--bg);\n  color: var(--text);\n  font: inherit;\n  font-size: 1rem;\n}\nform.stack input:focus { outline: 2px solid var(--accent); outline-offset: 1px; }\n'),
]

texts = {}
for rel, old, new in EDITS:
    text = texts.setdefault(rel, Path(rel).read_text())
    if text.count(old) != 1:
        sys.exit(f"ABORTED, nothing changed: {rel} does not match the expected 0.10.0 code "
                 f"(found {text.count(old)} matches for:\n{old})")
    texts[rel] = text.replace(old, new)
for rel, text in texts.items():
    shutil.copy2(rel, rel + ".bak")
    Path(rel).write_text(text)
    print(f"updated {rel}  (backup: {rel}.bak)")
print("Update to 0.11.0 complete.")
EOF
python3 update_to_0_11_0.py
```

**2. New file `app/auth.py`:**

```bash
cat > ~/surveillance/app/auth.py <<'EOF'
"""Accounts, password hashing, sessions, brute-force protection and the audit log (Phase 11).

Design:
  * Passwords are hashed with Argon2id (RFC 9106 low-memory profile: 64 MiB, 3 passes, 4 lanes).
  * At most two hashes are computed at once, so a flood of login attempts cannot exhaust RAM.
  * Unknown usernames are checked against a dummy hash, so response time does not reveal
    which usernames exist.
  * Failed logins lock the attempted username with a growing delay, whether or not the account
    exists (so the lockout message cannot reveal real usernames); a global cap limits guessing
    across all usernames. Counters are stored in the database, so a restart does not reset them.
  * Session tokens are 256-bit random values. Only their SHA-256 is stored, so a copy of the
    database does not contain usable sessions.
  * Sessions end after an idle period and after a maximum age; changing the password ends all
    other sessions.
"""
from __future__ import annotations

import hashlib
import logging
import os
import re
import secrets
import sqlite3
import threading
import time
from contextlib import closing
from pathlib import Path

from argon2 import PasswordHasher
from argon2.exceptions import InvalidHashError, VerificationError, VerifyMismatchError

log = logging.getLogger("Auth")

AUTH_DB_NAME = "auth.db"
MIN_PASSWORD_LENGTH = 15
MAX_PASSWORD_LENGTH = 256
USERNAME_RE = re.compile(r"^[a-z][a-z0-9_.-]{2,31}$")
TOKEN_BYTES = 32
MAX_SESSIONS_PER_USER = 10
SEEN_UPDATE_S = 60
GLOBAL_WINDOW_S = 600
GLOBAL_MAX_FAILURES = 30
LOCK_BASE_S = 30
LOCK_MAX_S = 900
HASH_SLOT_WAIT_S = 10
AUDIT_KEPT = 5000
LOCKOUT_FORGET_S = 86400

COMMON_PASSWORDS = {
    "password", "passwordpassword", "password123456", "123456789012345", "1234567890123456",
    "qwertyuiopasdfg", "qwertyuiopasdfgh", "iloveyouiloveyou", "adminadminadmin", "administrator123",
    "letmeinletmein", "welcomewelcome1", "raspberrypi1234", "raspberrypiraspberrypi", "changemechangeme",
    "correcthorsebatterystaple", "trustno1trustno1", "1q2w3e4r5t6y7u8i", "aaaaaaaaaaaaaaa",
}

SCHEMA = """
CREATE TABLE IF NOT EXISTS users (
    id                  INTEGER PRIMARY KEY,
    username            TEXT NOT NULL UNIQUE,
    password_hash       TEXT NOT NULL,
    created_at          INTEGER NOT NULL,
    password_changed_at INTEGER NOT NULL
);
CREATE TABLE IF NOT EXISTS sessions (
    token_hash  TEXT PRIMARY KEY,
    user_id     INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    csrf_token  TEXT NOT NULL,
    created_at  INTEGER NOT NULL,
    last_seen   INTEGER NOT NULL,
    reauth_at   INTEGER NOT NULL,
    user_agent  TEXT
);
CREATE INDEX IF NOT EXISTS idx_sessions_user ON sessions(user_id);
CREATE TABLE IF NOT EXISTS login_failures (at INTEGER NOT NULL);
-- Keyed by the attempted username, existing or not.
CREATE TABLE IF NOT EXISTS lockouts (
    username      TEXT PRIMARY KEY,
    failed        INTEGER NOT NULL,
    locked_until  INTEGER NOT NULL,
    updated_at    INTEGER NOT NULL
);
CREATE TABLE IF NOT EXISTS audit_log (
    id        INTEGER PRIMARY KEY,
    at        INTEGER NOT NULL,
    username  TEXT,
    action    TEXT NOT NULL,
    detail    TEXT
);
CREATE INDEX IF NOT EXISTS idx_audit_at ON audit_log(at);
"""


class AuthError(ValueError):
    pass


def hash_token(token: str) -> str:
    return hashlib.sha256(token.encode()).hexdigest()


def normalize_username(username: str) -> str:
    return username.strip().lower()


def safe_label(text: str) -> str:
    """For the audit log: keep attempted usernames short and printable."""
    return re.sub(r"[^a-z0-9_.@-]", "?", text.lower())[:32]


def password_problem(password: str, username: str) -> str | None:
    if len(password) < MIN_PASSWORD_LENGTH:
        return f"The password must be at least {MIN_PASSWORD_LENGTH} characters long (a passphrase works well)."
    if len(password) > MAX_PASSWORD_LENGTH:
        return f"The password must be at most {MAX_PASSWORD_LENGTH} characters long."
    lowered = password.lower()
    if lowered in COMMON_PASSWORDS or len(set(lowered)) < 5:
        return "That password is too easy to guess."
    if username and username.lower() in lowered:
        return "The password must not contain the username."
    return None


class AuthStore:
    def __init__(self, path: Path, idle_minutes: int, session_hours: int, lockout_threshold: int) -> None:
        self.path = path
        self._idle_s = idle_minutes * 60
        self._max_age_s = session_hours * 3600
        self._lock_threshold = lockout_threshold
        self._hasher = PasswordHasher()
        self._dummy_hash = self._hasher.hash(secrets.token_hex(16))
        self._hash_slots = threading.BoundedSemaphore(2)

    # ---------------------------------------------------------------- storage
    def _connect(self) -> sqlite3.Connection:
        conn = sqlite3.connect(self.path, timeout=5, isolation_level=None)
        conn.execute("PRAGMA journal_mode=WAL")
        conn.execute("PRAGMA synchronous=FULL")
        conn.execute("PRAGMA foreign_keys=ON")
        conn.execute("PRAGMA busy_timeout=5000")
        conn.row_factory = sqlite3.Row
        return conn

    def init(self) -> None:
        self.path.parent.mkdir(parents=True, exist_ok=True)
        old_umask = os.umask(0o077)  # auth.db and its WAL files are owner-only
        try:
            with closing(self._connect()) as conn:
                conn.executescript(SCHEMA)
        finally:
            os.umask(old_umask)
        os.chmod(self.path, 0o600)

    def audit(self, conn: sqlite3.Connection, username: str | None, action: str, detail: str = "") -> None:
        conn.execute("INSERT INTO audit_log (at, username, action, detail) VALUES (?, ?, ?, ?)",
                     (int(time.time()), username, action, detail[:200]))
        conn.execute("DELETE FROM audit_log WHERE id <= (SELECT MAX(id) FROM audit_log) - ?", (AUDIT_KEPT,))

    def record(self, username: str | None, action: str, detail: str = "") -> None:
        with closing(self._connect()) as conn:
            self.audit(conn, username, action, detail)

    # ---------------------------------------------------------------- hashing
    def _verify(self, password_hash: str, password: str) -> bool:
        if not self._hash_slots.acquire(timeout=HASH_SLOT_WAIT_S):
            raise AuthError("The camera is busy. Try again in a moment.")
        try:
            self._hasher.verify(password_hash, password)
            return True
        except (VerifyMismatchError, VerificationError, InvalidHashError):
            return False
        finally:
            self._hash_slots.release()

    # ------------------------------------------------------------------ users
    def user_count(self) -> int:
        with closing(self._connect()) as conn:
            return conn.execute("SELECT COUNT(*) FROM users").fetchone()[0]

    def list_users(self) -> list[sqlite3.Row]:
        with closing(self._connect()) as conn:
            return conn.execute("SELECT u.id, u.username, u.created_at, u.password_changed_at,"
                                " COALESCE(l.failed, 0) AS failed_logins, COALESCE(l.locked_until, 0) AS locked_until,"
                                " (SELECT COUNT(*) FROM sessions s WHERE s.user_id = u.id) AS sessions"
                                " FROM users u LEFT JOIN lockouts l ON l.username = u.username"
                                " ORDER BY u.username").fetchall()

    def create_user(self, username: str, password: str) -> None:
        username = normalize_username(username)
        if not USERNAME_RE.match(username):
            raise AuthError("Usernames are 3-32 characters: lowercase letters, digits, and . _ - (starting with a letter).")
        problem = password_problem(password, username)
        if problem:
            raise AuthError(problem)
        now = int(time.time())
        with closing(self._connect()) as conn:
            try:
                conn.execute("INSERT INTO users (username, password_hash, created_at, password_changed_at)"
                             " VALUES (?, ?, ?, ?)", (username, self._hasher.hash(password), now, now))
            except sqlite3.IntegrityError:
                raise AuthError(f"User {username} already exists.") from None
            self.audit(conn, username, "user_created")

    def set_password(self, username: str, password: str, actor: str = "cli") -> None:
        username = normalize_username(username)
        problem = password_problem(password, username)
        if problem:
            raise AuthError(problem)
        with closing(self._connect()) as conn:
            updated = conn.execute("UPDATE users SET password_hash = ?, password_changed_at = ? WHERE username = ?",
                                   (self._hasher.hash(password), int(time.time()), username)).rowcount
            if not updated:
                raise AuthError(f"No user called {username}.")
            conn.execute("DELETE FROM lockouts WHERE username = ?", (username,))
            conn.execute("DELETE FROM sessions WHERE user_id = (SELECT id FROM users WHERE username = ?)", (username,))
            self.audit(conn, username, "password_changed", f"by {actor}; all sessions ended")

    def unlock(self, username: str | None = None) -> None:
        with closing(self._connect()) as conn:
            if username:
                conn.execute("DELETE FROM lockouts WHERE username = ?", (normalize_username(username),))
            else:
                conn.execute("DELETE FROM lockouts")
            conn.execute("DELETE FROM login_failures")
            self.audit(conn, username, "unlocked", "by cli")

    # ------------------------------------------------------------------ login
    def authenticate(self, username: str, password: str, user_agent: str) -> tuple[str | None, str | None]:
        """Return (session_token, None) on success or (None, message for the user)."""
        name = normalize_username(username)[:64]
        password = password[:MAX_PASSWORD_LENGTH]
        now = int(time.time())
        with closing(self._connect()) as conn:
            recent = conn.execute("SELECT COUNT(*) FROM login_failures WHERE at > ?",
                                  (now - GLOBAL_WINDOW_S,)).fetchone()[0]
            if recent >= GLOBAL_MAX_FAILURES:
                self.audit(conn, safe_label(name), "login_blocked", "too many failures on all accounts")
                return None, "Too many failed logins recently. Try again in a few minutes."
            lock = conn.execute("SELECT * FROM lockouts WHERE username = ?", (name,)).fetchone()
            if lock and lock["locked_until"] > now:
                minutes = max(1, round((lock["locked_until"] - now) / 60))
                self.audit(conn, safe_label(name), "login_blocked", "username locked")
                return None, f"Too many failed attempts. Try again in about {minutes} minute(s)."
            user = conn.execute("SELECT * FROM users WHERE username = ?", (name,)).fetchone()

        try:
            ok = self._verify(user["password_hash"] if user else self._dummy_hash, password) and user is not None
        except AuthError as exc:
            return None, str(exc)

        with closing(self._connect()) as conn:
            if not ok:
                conn.execute("INSERT INTO login_failures (at) VALUES (?)", (now,))
                conn.execute("DELETE FROM login_failures WHERE at <= ?", (now - GLOBAL_WINDOW_S,))
                failed = (lock["failed"] if lock else 0) + 1
                locked_until = 0
                if failed >= self._lock_threshold:
                    locked_until = now + min(LOCK_MAX_S, LOCK_BASE_S * 2 ** (failed - self._lock_threshold))
                conn.execute("INSERT INTO lockouts (username, failed, locked_until, updated_at) VALUES (?, ?, ?, ?)"
                             " ON CONFLICT(username) DO UPDATE SET failed = excluded.failed,"
                             " locked_until = excluded.locked_until, updated_at = excluded.updated_at",
                             (name, failed, locked_until, now))
                conn.execute("DELETE FROM lockouts WHERE updated_at < ? AND locked_until < ?",
                             (now - LOCKOUT_FORGET_S, now))
                self.audit(conn, safe_label(name), "login_failed")
                log.warning("Failed login for %r", safe_label(name))
                return None, "Wrong username or password."

            conn.execute("DELETE FROM lockouts WHERE username = ?", (name,))
            if self._hasher.check_needs_rehash(user["password_hash"]):
                conn.execute("UPDATE users SET password_hash = ? WHERE id = ?",
                             (self._hasher.hash(password), user["id"]))
            token = self._new_session(conn, user["id"], user_agent)
            self.audit(conn, name, "login")
            log.info("User %s logged in", name)
            return token, None

    def _new_session(self, conn: sqlite3.Connection, user_id: int, user_agent: str) -> str:
        token = secrets.token_urlsafe(TOKEN_BYTES)
        now = int(time.time())
        conn.execute("INSERT INTO sessions (token_hash, user_id, csrf_token, created_at, last_seen, reauth_at,"
                     " user_agent) VALUES (?, ?, ?, ?, ?, ?, ?)",
                     (hash_token(token), user_id, secrets.token_urlsafe(TOKEN_BYTES), now, now, now,
                      user_agent[:200]))
        conn.execute("DELETE FROM sessions WHERE user_id = ? AND token_hash NOT IN (SELECT token_hash FROM sessions"
                     " WHERE user_id = ? ORDER BY created_at DESC LIMIT ?)", (user_id, user_id, MAX_SESSIONS_PER_USER))
        return token

    # --------------------------------------------------------------- sessions
    def get_session(self, token: str) -> dict | None:
        if not token or len(token) > 100:
            return None
        now = int(time.time())
        with closing(self._connect()) as conn:
            row = conn.execute("SELECT s.*, u.username FROM sessions s JOIN users u ON u.id = s.user_id"
                               " WHERE s.token_hash = ?", (hash_token(token),)).fetchone()
            if row is None:
                return None
            if now - row["last_seen"] > self._idle_s or now - row["created_at"] > self._max_age_s:
                conn.execute("DELETE FROM sessions WHERE token_hash = ?", (row["token_hash"],))
                return None
            if now - row["last_seen"] >= SEEN_UPDATE_S:
                conn.execute("UPDATE sessions SET last_seen = ? WHERE token_hash = ?", (now, row["token_hash"]))
            return dict(row)

    def logout(self, session: dict) -> None:
        with closing(self._connect()) as conn:
            conn.execute("DELETE FROM sessions WHERE token_hash = ?", (session["token_hash"],))
            self.audit(conn, session["username"], "logout")

    def logout_others(self, session: dict) -> int:
        with closing(self._connect()) as conn:
            ended = conn.execute("DELETE FROM sessions WHERE user_id = ? AND token_hash != ?",
                                 (session["user_id"], session["token_hash"])).rowcount
            self.audit(conn, session["username"], "logout_others", f"{ended} session(s) ended")
            return ended

    def logout_all(self, username: str) -> int:
        with closing(self._connect()) as conn:
            ended = conn.execute("DELETE FROM sessions WHERE user_id = (SELECT id FROM users WHERE username = ?)",
                                 (normalize_username(username),)).rowcount
            self.audit(conn, normalize_username(username), "logout_all", f"by cli; {ended} session(s) ended")
            return ended

    def sessions_for(self, user_id: int) -> list[sqlite3.Row]:
        with closing(self._connect()) as conn:
            return conn.execute("SELECT token_hash, created_at, last_seen, user_agent FROM sessions"
                                " WHERE user_id = ? ORDER BY last_seen DESC", (user_id,)).fetchall()

    def change_password(self, session: dict, current: str, new: str) -> None:
        with closing(self._connect()) as conn:
            user = conn.execute("SELECT * FROM users WHERE id = ?", (session["user_id"],)).fetchone()
        if not self._verify(user["password_hash"], current[:MAX_PASSWORD_LENGTH]):
            self.record(user["username"], "password_change_failed", "wrong current password")
            raise AuthError("The current password is wrong.")
        problem = password_problem(new, user["username"])
        if problem:
            raise AuthError(problem)
        with closing(self._connect()) as conn:
            conn.execute("UPDATE users SET password_hash = ?, password_changed_at = ? WHERE id = ?",
                         (self._hasher.hash(new), int(time.time()), user["id"]))
            conn.execute("DELETE FROM sessions WHERE user_id = ? AND token_hash != ?",
                         (user["id"], session["token_hash"]))
            self.audit(conn, user["username"], "password_changed", "by web; other sessions ended")

    def recent_audit(self, limit: int = 20) -> list[sqlite3.Row]:
        with closing(self._connect()) as conn:
            return conn.execute("SELECT * FROM audit_log ORDER BY id DESC LIMIT ?", (limit,)).fetchall()
EOF
```

**3. New file `app/web_auth.py`:**

```bash
cat > ~/surveillance/app/web_auth.py <<'EOF'
"""Login, logout, account page and the checks that protect every other page (Phase 11).

Every request except the login page and static files needs a valid session:
  * pages redirect to /login; API, video, download and live endpoints answer 401
  * every POST must come from this site (Origin/Referer check) and carry the session's
    CSRF token, so another website cannot make your browser perform actions here
  * the session cookie is HttpOnly (invisible to JavaScript), Secure, SameSite=Strict and
    uses the __Host- prefix, so it is never sent to other sites or sub-domains
"""
from __future__ import annotations

import hmac
import logging
from datetime import datetime
from urllib.parse import urlsplit

from flask import Blueprint, abort, g, redirect, render_template, request, url_for

from app.auth import AUTH_DB_NAME, AuthError, AuthStore
from app.config import Settings

log = logging.getLogger("Auth")

COOKIE_NAME = "__Host-sid"
PUBLIC_ENDPOINTS = {"auth.login", "auth.login_post", "static"}
RESOURCE_ENDPOINTS = {"api_status", "live.stream", "live.snapshot", "rec.video", "rec.download"}
SAFE_METHODS = {"GET", "HEAD", "OPTIONS"}
MESSAGES = {
    "password_changed": "Password changed. All your other sessions were logged out.",
    "others_logged_out": "All other sessions were logged out.",
    "logged_out": "You have been logged out.",
    "expired": "Please log in.",
}


def safe_next(target: str | None) -> str:
    """Only allow redirects to paths on this site (never //other.example or https://...)."""
    if not target or not target.startswith("/") or target.startswith("//") or "\\" in target:
        return url_for("dashboard")
    return target


def same_origin() -> bool:
    for header in ("Origin", "Referer"):
        value = request.headers.get(header)
        if value:
            return value != "null" and urlsplit(value).netloc == request.host
    return False


def install(app, settings: Settings) -> AuthStore:
    auth = settings.auth
    store = AuthStore(settings.database_dir / AUTH_DB_NAME, auth.idle_timeout_minutes,
                      auth.session_max_hours, auth.lockout_threshold)
    store.init()
    bp = Blueprint("auth", __name__)

    @app.before_request
    def require_login():
        g.session = None
        token = request.cookies.get(COOKIE_NAME)
        if token:
            g.session = store.get_session(token)
        if request.method not in SAFE_METHODS and not same_origin():
            abort(403)
        if request.endpoint in PUBLIC_ENDPOINTS:
            return None
        if g.session is None:
            if request.endpoint in RESOURCE_ENDPOINTS or request.endpoint is None:
                abort(401)
            return redirect(url_for("auth.login", next=request.full_path.rstrip("?"), msg="expired"))
        if request.method not in SAFE_METHODS:
            sent = request.form.get("csrf_token") or request.headers.get("X-CSRF-Token", "")
            if not hmac.compare_digest(sent, g.session["csrf_token"]):
                abort(403)
        return None

    @app.context_processor
    def auth_context() -> dict:
        session = getattr(g, "session", None)
        return {"current_user": session["username"] if session else None,
                "csrf_token": session["csrf_token"] if session else ""}

    @app.errorhandler(401)
    def unauthorized(_error):
        return "Login required", 401

    @app.errorhandler(403)
    def forbidden(_error):
        return render_template("placeholder.html", title="Forbidden", page="", phase=None,
                               text="This request was refused (it did not come from this page, or the form "
                                    "expired). Go back, reload the page and try again."), 403

    def set_session_cookie(response, token: str):
        response.set_cookie(COOKIE_NAME, token, httponly=True, secure=True, samesite="Strict", path="/")
        return response

    @bp.get("/login")
    def login():
        if g.session:
            return redirect(safe_next(request.args.get("next")))
        return render_template("login.html", title="Log in", page="", next=request.args.get("next", ""),
                               error=None, username="", message=MESSAGES.get(request.args.get("msg", "")),
                               no_users=store.user_count() == 0)

    @bp.post("/login")
    def login_post():
        username = request.form.get("username", "")[:64]
        password = request.form.get("password", "")
        target = request.form.get("next", "")
        token, error = store.authenticate(username, password, request.headers.get("User-Agent", ""))
        if error:
            return render_template("login.html", title="Log in", page="", next=target, error=error,
                                   username=username, message=None, no_users=store.user_count() == 0), 401
        return set_session_cookie(redirect(safe_next(target)), token)

    @bp.post("/logout")
    def logout():
        store.logout(g.session)
        response = redirect(url_for("auth.login", msg="logged_out"))
        response.delete_cookie(COOKIE_NAME, path="/", secure=True, httponly=True, samesite="Strict")
        return response

    @bp.get("/account")
    def account():
        sessions = [{"current": s["token_hash"] == g.session["token_hash"],
                     "created": datetime.fromtimestamp(s["created_at"]),
                     "last_seen": datetime.fromtimestamp(s["last_seen"]),
                     "agent": (s["user_agent"] or "unknown browser")[:80]}
                    for s in store.sessions_for(g.session["user_id"])]
        audit = [{"at": datetime.fromtimestamp(a["at"]), "username": a["username"] or "",
                  "action": a["action"].replace("_", " "), "detail": a["detail"] or ""}
                 for a in store.recent_audit(25)]
        return render_template("account.html", title="Account", page="auth.account", sessions=sessions,
                               audit=audit, error=None, message=MESSAGES.get(request.args.get("msg", "")))

    @bp.post("/account/password")
    def change_password():
        new, confirm = request.form.get("new_password", ""), request.form.get("confirm_password", "")
        try:
            if new != confirm:
                raise AuthError("The two new passwords do not match.")
            store.change_password(g.session, request.form.get("current_password", ""), new)
        except AuthError as exc:
            return render_template("account.html", title="Account", page="auth.account", sessions=[], audit=[],
                                   error=str(exc), message=None), 400
        return redirect(url_for("auth.account", msg="password_changed"))

    @bp.post("/account/logout-others")
    def logout_others():
        store.logout_others(g.session)
        return redirect(url_for("auth.account", msg="others_logged_out"))

    app.register_blueprint(bp)
    return store
EOF
```

**4. New templates, `login.html` and `account.html`:**

```bash
cat > ~/surveillance/web/templates/login.html <<'EOF'
{% extends "base.html" %}
{% block content %}
<section class="panel login-panel">
  <h2>Log in</h2>
  {% if message %}<p class="notice">{{ message }}</p>{% endif %}
  {% if no_users %}
  <p class="notice">No account exists yet. Create one on the Raspberry Pi (over SSH):<br>
    <code>cd ~/surveillance &amp;&amp; python3 tools/manage_users.py create admin</code></p>
  {% endif %}
  <form method="post" action="{{ url_for('auth.login_post') }}" class="stack">
    <input type="hidden" name="next" value="{{ next }}">
    <label>Username
      <input type="text" name="username" value="{{ username }}" autocomplete="username" autocapitalize="none"
             spellcheck="false" maxlength="64" required autofocus>
    </label>
    <label>Password
      <input type="password" name="password" autocomplete="current-password" maxlength="256" required>
    </label>
    {% if error %}<p class="error" role="alert">{{ error }}</p>{% endif %}
    <button class="button primary" type="submit">Log in</button>
  </form>
</section>
{% endblock %}
EOF

cat > ~/surveillance/web/templates/account.html <<'EOF'
{% extends "base.html" %}
{% block content %}
{% if message %}<p class="notice">{{ message }}</p>{% endif %}

<section class="panel">
  <h2>Change password</h2>
  <form method="post" action="{{ url_for('auth.change_password') }}" class="stack narrow">
    <input type="hidden" name="csrf_token" value="{{ csrf_token }}">
    <label>Current password <input type="password" name="current_password" autocomplete="current-password" maxlength="256" required></label>
    <label>New password (at least 15 characters) <input type="password" name="new_password" autocomplete="new-password" minlength="15" maxlength="256" required></label>
    <label>Repeat new password <input type="password" name="confirm_password" autocomplete="new-password" minlength="15" maxlength="256" required></label>
    {% if error %}<p class="error" role="alert">{{ error }}</p>{% endif %}
    <button class="button primary" type="submit">Change password</button>
  </form>
  <p class="muted">A long passphrase from a password manager is best. Changing it logs out every other session.</p>
</section>

{% if sessions %}
<section class="panel">
  <h2>Active sessions</h2>
  <div class="table-wrap">
    <table class="list">
      <thead><tr><th>Browser</th><th>Logged in</th><th>Last active</th><th></th></tr></thead>
      <tbody>
      {% for s in sessions %}
        <tr><td>{{ s.agent }}</td><td>{{ s.created.strftime('%Y-%m-%d %H:%M') }}</td>
          <td>{{ s.last_seen.strftime('%Y-%m-%d %H:%M') }}</td>
          <td>{% if s.current %}<span class="badge motion">this browser</span>{% endif %}</td></tr>
      {% endfor %}
      </tbody>
    </table>
  </div>
  <form method="post" action="{{ url_for('auth.logout_others') }}" class="player-actions">
    <input type="hidden" name="csrf_token" value="{{ csrf_token }}">
    <button class="button" type="submit">Log out all other sessions</button>
  </form>
</section>
{% endif %}

{% if audit %}
<section class="panel">
  <h2>Security log (latest 25)</h2>
  <div class="table-wrap">
    <table class="list">
      <thead><tr><th>Time</th><th>User</th><th>Event</th><th>Detail</th></tr></thead>
      <tbody>
      {% for a in audit %}
        <tr><td>{{ a.at.strftime('%Y-%m-%d %H:%M:%S') }}</td><td>{{ a.username }}</td>
          <td>{{ a.action }}</td><td>{{ a.detail }}</td></tr>
      {% endfor %}
      </tbody>
    </table>
  </div>
</section>
{% endif %}
{% endblock %}
EOF
```

**5. New tool `tools/manage_users.py`:**

```bash
cat > ~/surveillance/tools/manage_users.py <<'EOF'
#!/usr/bin/env python3
"""Manage web accounts (run on the Pi over SSH; there is deliberately no web sign-up page).

    python3 ~/surveillance/tools/manage_users.py create admin     # asks for the password twice
    python3 ~/surveillance/tools/manage_users.py passwd admin     # set a new password, ends all sessions
    python3 ~/surveillance/tools/manage_users.py list
    python3 ~/surveillance/tools/manage_users.py unlock [admin]   # clear failed-login lockouts
    python3 ~/surveillance/tools/manage_users.py logout admin     # end every session of the user
    python3 ~/surveillance/tools/manage_users.py log              # recent security events
"""
from __future__ import annotations

import argparse
import getpass
import sys
from datetime import datetime
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent.parent))

from app.auth import AUTH_DB_NAME, AuthError, AuthStore  # noqa: E402
from app.config import DEFAULT_CONFIG_PATH, ConfigError, load_settings  # noqa: E402


def ask_password(username: str) -> str:
    print("Choose a password of at least 15 characters. A passphrase of 4-5 random words works well;")
    print("a password manager can generate and remember one for you.")
    first = getpass.getpass(f"New password for {username}: ")
    second = getpass.getpass("Repeat it: ")
    if first != second:
        raise AuthError("The two passwords do not match.")
    return first


def when(epoch: int) -> str:
    return datetime.fromtimestamp(epoch).strftime("%Y-%m-%d %H:%M") if epoch else "-"


def main() -> int:
    parser = argparse.ArgumentParser(description="Manage surveillance web accounts")
    parser.add_argument("--config", type=Path, default=DEFAULT_CONFIG_PATH)
    sub = parser.add_subparsers(dest="command", required=True)
    for name in ("create", "passwd", "logout"):
        sub.add_parser(name).add_argument("username")
    sub.add_parser("unlock").add_argument("username", nargs="?")
    sub.add_parser("list")
    sub.add_parser("log")
    args = parser.parse_args()

    try:
        settings, _ = load_settings(args.config, create_if_missing=False)
    except ConfigError as exc:
        print(f"Configuration error: {exc}", file=sys.stderr)
        return 2
    auth = settings.auth
    store = AuthStore(settings.database_dir / AUTH_DB_NAME, auth.idle_timeout_minutes,
                      auth.session_max_hours, auth.lockout_threshold)
    store.init()

    try:
        if args.command == "create":
            store.create_user(args.username, ask_password(args.username.lower()))
            print(f"User {args.username.lower()} created. Log in at http://localhost:8080/login")
        elif args.command == "passwd":
            store.set_password(args.username, ask_password(args.username.lower()))
            print("Password changed; every session of this user was logged out.")
        elif args.command == "logout":
            print(f"{store.logout_all(args.username)} session(s) ended.")
        elif args.command == "unlock":
            store.unlock(args.username)
            print("Lockouts cleared.")
        elif args.command == "list":
            users = store.list_users()
            if not users:
                print("No users yet. Create one with: python3 tools/manage_users.py create admin")
            for u in users:
                locked = " LOCKED until " + when(u["locked_until"]) if u["locked_until"] > datetime.now().timestamp() else ""
                print(f"{u['username']:<20} created {when(u['created_at'])}  password changed "
                      f"{when(u['password_changed_at'])}  sessions {u['sessions']}  failed logins "
                      f"{u['failed_logins']}{locked}")
        elif args.command == "log":
            for a in reversed(store.recent_audit(50)):
                print(f"{when(a['at'])}  {a['username'] or '-':<20} {a['action']:<24} {a['detail'] or ''}")
    except AuthError as exc:
        print(f"Error: {exc}", file=sys.stderr)
        return 1
    return 0


if __name__ == "__main__":
    sys.exit(main())
EOF
```

## Test procedure

**1. Create your account on the Pi.** The password won't be shown as you type:

```bash
cd ~/surveillance
python3 tools/manage_users.py create admin
python3 tools/manage_users.py list
ls -l database/auth.db          # must show -rw------- (only you can read it)
```

**2. Restart the web interface** (Ctrl+C in window 2, then `python3 -m app.web_main`). The recorder can keep running. Use **Chrome, Edge or Firefox** on the laptop, through the tunnel as before.

**3. In the browser:**
- Open **http://localhost:8080**. You should land on the **Log in** page, with no navigation bar.
- Log in with a **wrong** password. Expect "Wrong username or password."
- Log in correctly. Every page works as before. **Live View** and playing a recording should work too, since they're protected by the same session.
- Your username and **Log out** appear on the right of the navigation bar.

**4. Account page** (click your username):
- **Change password.** First enter a wrong current password (it should be refused), then do it properly.
- **Sessions:** log in from a second browser or a private window, check it appears under **Active sessions**, then use **Log out all other sessions**. The other browser should land on the login page at its next click, or within about 5 s on the dashboard.

**5. Lockout.**
1. Log out, then enter a wrong password **6 times**. From the 6th attempt you'll see "Too many failed attempts".
2. Check that it's locked, then unlock it:
   ```bash
   python3 tools/manage_users.py list      # shows LOCKED until …
   python3 tools/manage_users.py unlock admin
   ```
3. Log in normally.

**6. Security checks, on the Pi, while logged out:**

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8080/api/status            # 401
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8080/live/stream           # 401
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8080/recordings/1/video    # 401
curl -s -o /dev/null -w "%{http_code}\n" -X POST -H "Origin: https://evil.example" http://127.0.0.1:8080/login   # 403
python3 tools/manage_users.py log | tail -10
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| Logging in just shows the login page again, with no error | The browser refused the secure cookie; this is the case in Safari. Use Chrome, Edge or Firefox through the tunnel. |
| "This request was refused" | You submitted a form from a page opened before restarting the web interface, or from another site. Reload the page and try again. |
| You locked yourself out | Run `python3 tools/manage_users.py unlock` on the Pi. |
| You forgot your password | Run `python3 tools/manage_users.py passwd admin` on the Pi. |
| `ModuleNotFoundError: argon2` | `sudo apt install -y python3-argon2` |

**Please send me:**
- whether login, logout, the password change and the lockout behaved as described;
- the output of step 6;
- a screenshot of the Account page, if you like.

Next is **Phase 11b**, the editable Settings page:
- every setting validated;
- dangerous changes (for example the storage limit or the recordings folder) require re-entering your password;
- the recorder picks up changes automatically;
- the optional `camera.tuning_file` for your NoIR camera's colours.

After that, Phase 12 sets up Tailscale for private remote access.
