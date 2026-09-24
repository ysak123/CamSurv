Phase 13b is confirmed on the real Pi. Phase 14 (hardening) is ready as version 0.14.0, in four parts that you run one at a time:

| Part | What it does | Lock-out protection |
|---|---|---|
| **SSH** (`harden.sh ssh`) | Keys only (no passwords), no root login, only your account, only `-L` forwarding (so the SSH tunnel keeps working) | Refuses until your key is installed **and** the log shows you've logged in with it at least once |
| **Firewall** (`harden.sh firewall`) | Nothing on the home network or internet can open a connection to the Pi. Only tailnet devices reach SSH (22) and the web page (443). Ping, DHCP, Tailscale's own traffic, and connections the Pi opens itself (updates, time) keep working | Refuses if Tailscale is down, and removes itself after 3 minutes unless you confirm from a **new** SSH session over the tailnet. It uses its own rules table, so Tailscale's rules are left alone |
| **Updates** (`harden.sh updates`) | Daily automatic Debian stable and security updates | Camera packages from the Raspberry Pi archive (kernel, firmware, libcamera, picamera2) stay manual, so an update can't change the camera without you knowing |
| **Audit** (`phase14_security_audit.py`) | Read-only check of what's really in effect: sshd's effective settings, keys, the firewall, listening ports, sudo rules, updates, service sandboxes, file permissions and accounts | Changes nothing |

I tested all four on the Debian 13 test system, and the firewall rules in an isolated network namespace as well:

- **SSH:** `harden.sh ssh` refused with no key, and refused again with a key that had never been used. After a key login it applied the settings, and they won even over a Pi-style `50-cloud-init.conf` that says `PasswordAuthentication yes`.
  - Key login is accepted.
  - Password login and root login (even with a valid key) are refused.
  - `-L` to the web page still works; `-R` and agent forwarding are refused.
- **Firewall:** from the home network, SSH, 443 and 8080 are blocked over both IPv4 and IPv6. From `tailscale0`, 22 and 443 are open and 8080 isn't. Ping, Tailscale's traffic and the Pi's own outgoing connections work.
  - Unconfirmed, it removed itself after the delay. Confirmed, it survived a reboot and loaded before the network.
- **Updates and audit:** automatic updates were installed and switched on. The audit ended with 37 pass, 0 fail, and only the two warnings I expected from the test system.

## Step 1 — Apply the update and deploy it

```bash
cd ~/surveillance
cat > update_to_0_14_0.py <<'PYEOF'
#!/usr/bin/env python3
"""Update the surveillance project from 0.13.2 to 0.14.0 (run from ~/surveillance)."""
import os, shutil, sys
from pathlib import Path

EDITS = [
    ('app/__init__.py',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.13.2"\n',
     '"""Raspberry Pi surveillance camera."""\n\n__version__ = "0.14.0"\n'),
    ('tools/phase12_tailscale_check.py',
     'HTTPS_TIMEOUT_S = 60\nKNOWN_PORTS = {\n    22: "SSH, restricted to the tailnet in Phase 14",\n    111: "rpcbind (an NFS helper this camera does not need): disable it, Phase 12 step 13",\n}\n',
     'HTTPS_TIMEOUT_S = 60\nKNOWN_PORTS = {\n    22: "SSH: limit it to the tailnet with the Phase 14 firewall",\n    111: "rpcbind (an NFS helper this camera does not need): disable it, Phase 12 step 13",\n}\n'),
    ('tools/phase12_tailscale_check.py',
     '\n    print("Listening ports")\n    sockets = listening_tcp()\n    web = [s for s in sockets if s[1] == web_port]\n',
     '\n    print("Listening ports")\n    firewall = subprocess.run(["systemctl", "is-active", "--quiet", "surveillance-firewall"],\n                              check=False).returncode == 0\n    sockets = listening_tcp()\n    web = [s for s in sockets if s[1] == web_port]\n'),
    ('tools/phase12_tailscale_check.py',
     '        if is_tailnet(addr):\n            print(f"         tailnet only : {addr}:{port}{who}")\n        else:\n            label = KNOWN_PORTS.get(port, "check what this is")\n',
     '        if is_tailnet(addr):\n            print(f"         tailnet only : {addr}:{port}{who}")\n        elif firewall:\n            report("PASS", f"{addr or \'*\'}:{port}{who}: the firewall blocks it from the home network"\n                           + (" (open inside the tailnet)" if port in (22, 443) else ""))\n        else:\n            label = KNOWN_PORTS.get(port, "check what this is")\n'),
]

NEW_FILES = [
    ('deploy/harden.sh', 0o755,
     '#!/bin/bash\n# Phase 14: harden the Pi. Run each part on its own, from your project folder:\n#\n#   sudo bash ~/surveillance/deploy/harden.sh ssh               SSH: keys only, no passwords, no root\n#   sudo bash ~/surveillance/deploy/harden.sh firewall          firewall ON, switches itself off after\n#                                                                3 minutes unless confirmed\n#   sudo bash ~/surveillance/deploy/harden.sh firewall-confirm  keep it (from a NEW SSH session over the tailnet)\n#   sudo bash ~/surveillance/deploy/harden.sh firewall-off      remove it again\n#   sudo bash ~/surveillance/deploy/harden.sh updates           automatic security updates\n#\n# Each part refuses to run if it could lock you out (no working SSH key yet, Tailscale down).\nset -euo pipefail\n\nAPP=/opt/surveillance\nSSHD_DROPIN=/etc/ssh/sshd_config.d/10-surveillance.conf\nFIREWALL_UNIT=surveillance-firewall.service\nUNDO_UNIT=surveillance-firewall-undo\nUNDO_AFTER_S=180\n\nsay() { printf \'\\n== %s\\n\' "$*"; }\ndie() { printf \'\\nERROR: %s\\n\' "$*" >&2; exit 1; }\n\n[ "$(id -u)" -eq 0 ] || die "run it with sudo: sudo bash $0 ${1:-}"\n[ -f "$APP/deploy/firewall.nft" ] || die "$APP is missing or old: run sudo bash ~/surveillance/deploy/install.sh first"\nADMIN="${SUDO_USER:-}"\n\nharden_ssh() {\n    [[ "$ADMIN" =~ ^[a-z_][a-z0-9_-]{0,31}$ ]] && [ "$ADMIN" != root ] || \\\n        die "run this with sudo from your own account (not as root)"\n    local home keys\n    home="$(getent passwd "$ADMIN" | cut -d: -f6)"\n    keys="$home/.ssh/authorized_keys"\n    say "Checking that $ADMIN can log in with a key"\n    grep -Eqs \'^(ssh-ed25519|ecdsa-sha2-|sk-ssh-ed25519|sk-ecdsa-sha2-|ssh-rsa) \' "$keys" || \\\n        die "no SSH public key in $keys yet. Add your laptop\'s key first (Phase 14, step 1)."\n    echo "keys in $keys: $(grep -Ec \'^(ssh-|ecdsa-|sk-)\' "$keys")"\n    if ! journalctl --since "-7 days" -o cat -t sshd -t sshd-session 2>/dev/null | \\\n            grep -q "Accepted publickey for $ADMIN "; then\n        die "no successful key login for $ADMIN in the last 7 days. Log in once with your key (step 1), then run this again."\n    fi\n    echo "a key login for $ADMIN was seen"\n    local mode\n    mode="$(stat -c %a "$home/.ssh")"\n    [ "$mode" = 700 ] || { chmod 700 "$home/.ssh"; echo "fixed permissions of $home/.ssh (was $mode)"; }\n    chmod 600 "$keys"\n\n    say "Installing $SSHD_DROPIN"\n    sed "s/^AllowUsers ADMIN_USER$/AllowUsers $ADMIN/" "$APP/deploy/sshd-hardening.conf" > "$SSHD_DROPIN.new"\n    chmod 0644 "$SSHD_DROPIN.new"\n    mv "$SSHD_DROPIN.new" "$SSHD_DROPIN"\n    if ! sshd -t; then\n        rm -f "$SSHD_DROPIN"\n        die "sshd rejected the new settings; they were removed again, nothing changed"\n    fi\n    systemctl reload ssh.service 2>/dev/null || systemctl reload sshd.service\n    echo "sshd reloaded (your current session stays open)"\n    say "Settings sshd now uses"\n    sshd -T | grep -E \'^(pubkeyauthentication|authenticationmethods|passwordauthentication|kbdinteractiveauthentication|permitrootlogin|allowusers|allowtcpforwarding|x11forwarding|maxauthtries) \' | sed \'s/^/  /\'\n    cat <<EOF\n\nKeep this session open. In a NEW terminal on your laptop, check:\n  ssh $ADMIN@cam01                                  -> logs in with your key\n  ssh -o PubkeyAuthentication=no $ADMIN@cam01       -> "Permission denied (publickey)"\nIf the first one fails, undo it from this session:  sudo rm $SSHD_DROPIN && sudo systemctl reload ssh\nEOF\n}\n\nfirewall_on() {\n    command -v nft >/dev/null || { say "Installing nftables"; apt-get install -y nftables; }\n    say "Checking that Tailscale is up (the only way in once the firewall is on)"\n    ip link show tailscale0 >/dev/null 2>&1 || die "there is no tailscale0 interface: is Tailscale running?"\n    tailscale status --json 2>/dev/null | grep -q \'"BackendState": "Running"\' || die "Tailscale is not connected"\n    echo "Tailscale is connected"\n    if systemctl is-enabled nftables.service >/dev/null 2>&1; then\n        echo "WARNING: nftables.service is enabled. Its /etc/nftables.conf usually starts with \'flush ruleset\',"\n        echo "         which also removes Tailscale\'s own rules. Disable it: sudo systemctl disable nftables"\n    fi\n    nft -c -f "$APP/deploy/firewall.nft" || die "the firewall rules do not load; nothing changed"\n\n    say "Firewall ON (for $UNDO_AFTER_S seconds unless you confirm)"\n    install -m 0644 "$APP/deploy/$FIREWALL_UNIT" /etc/systemd/system/\n    systemctl daemon-reload\n    systemctl stop "$UNDO_UNIT.timer" 2>/dev/null || true\n    systemctl reset-failed "$UNDO_UNIT.service" "$UNDO_UNIT.timer" 2>/dev/null || true\n    systemctl restart "$FIREWALL_UNIT"\n    systemd-run --quiet --unit="$UNDO_UNIT" --on-active="$UNDO_AFTER_S" --timer-property=AccuracySec=1s \\\n        systemctl stop "$FIREWALL_UNIT"\n    nft list chain inet cctv_filter input | sed \'s/^/  /\'\n    cat <<EOF\n\nIt switches itself off at $(date -d "+$UNDO_AFTER_S seconds" +%H:%M:%S) unless you confirm. Now, in a NEW terminal on your laptop:\n  ssh $ADMIN@cam01                                                   (over the tailnet)\n  sudo bash ~/surveillance/deploy/harden.sh firewall-confirm\nIf that new connection does not work, do nothing: the firewall removes itself.\nEOF\n}\n\nfirewall_confirm() {\n    systemctl is-active --quiet "$FIREWALL_UNIT" || \\\n        die "the firewall is not running (it may have switched itself off already). Run \'firewall\' again."\n    systemctl stop "$UNDO_UNIT.timer" 2>/dev/null || true\n    systemctl enable "$FIREWALL_UNIT"\n    say "Firewall kept, and on at every boot"\n    echo "Emergency off (keyboard and screen on the Pi): sudo systemctl disable --now $FIREWALL_UNIT"\n}\n\nfirewall_off() {\n    systemctl stop "$UNDO_UNIT.timer" 2>/dev/null || true\n    systemctl disable --now "$FIREWALL_UNIT" 2>/dev/null || true\n    nft delete table inet cctv_filter 2>/dev/null || true\n    say "Firewall removed"\n}\n\nauto_updates() {\n    say "Installing unattended-upgrades"\n    DEBIAN_FRONTEND=noninteractive apt-get install -y unattended-upgrades\n    install -m 0644 "$APP/deploy/apt-auto-upgrades.conf" /etc/apt/apt.conf.d/20auto-upgrades\n    systemctl enable --now apt-daily.timer apt-daily-upgrade.timer\n    say "Configuration"\n    apt-config dump | grep -E \'^APT::Periodic::(Update-Package-Lists|Unattended-Upgrade) \' | sed \'s/^/  /\'\n    apt-config dump | grep -E \'^Unattended-Upgrade::Origins-Pattern::\' | sed \'s/^/  /\'\n    systemctl list-timers apt-daily.timer apt-daily-upgrade.timer --no-pager | sed \'s/^/  /\'\n    say "Trial run (installs nothing)"\n    unattended-upgrade --dry-run -v 2>&1 | tail -n 5 | sed \'s/^/  /\' || true\n    echo "Log of real runs: /var/log/unattended-upgrades/unattended-upgrades.log"\n}\n\ncase "${1:-}" in\n    ssh) harden_ssh ;;\n    firewall) firewall_on ;;\n    firewall-confirm) firewall_confirm ;;\n    firewall-off) firewall_off ;;\n    updates) auto_updates ;;\n    *) sed -n \'2,11p\' "$0" | sed \'s/^# \\{0,1\\}//\'; exit 2 ;;\nesac\n'),
    ('deploy/firewall.nft', 0o644,
     '#!/usr/sbin/nft -f\n# Phase 14 firewall, loaded by surveillance-firewall.service (deploy/harden.sh installs it).\n#\n# Nothing on the home network or the internet can open a connection to the Pi. Only devices in\n# your tailnet reach SSH (22) and the camera page (443, tailscale serve), and the Tailscale\n# access policy narrows that to your own devices. Connections the Pi opens itself (updates,\n# time, Tailscale) and their answers are not affected.\n#\n# This is its own table: Tailscale keeps its own firewall rules, and they are left alone.\n\ntable inet cctv_filter\ndelete table inet cctv_filter\n\ntable inet cctv_filter {\n    chain input {\n        type filter hook input priority filter; policy drop;\n\n        iif "lo" accept\n        ct state established,related accept\n        ct state invalid drop\n\n        # Error messages both IP versions need, IPv6 neighbour discovery, and rate-limited ping.\n        icmp type { destination-unreachable, time-exceeded, parameter-problem } accept\n        icmpv6 type { destination-unreachable, packet-too-big, time-exceeded, parameter-problem,\n                      nd-router-advert, nd-neighbor-solicit, nd-neighbor-advert } accept\n        icmp type echo-request limit rate 5/second accept\n        icmpv6 type echo-request limit rate 5/second accept\n\n        # Address assignment from the router (DHCP, DHCPv6).\n        udp sport 67 udp dport 68 accept\n        udp sport 547 udp dport 546 accept\n\n        # Tailscale\'s encrypted WireGuard packets (direct connections between your devices).\n        udp dport 41641 accept\n\n        # Inside the tailnet only: SSH and the camera web page.\n        iifname "tailscale0" tcp dport { 22, 443 } accept\n\n        counter comment "dropped"\n    }\n\n    chain forward {\n        type filter hook forward priority filter; policy drop;\n    }\n}\n'),
    ('deploy/surveillance-firewall.service', 0o644,
     '# Phase 14 firewall (deploy/firewall.nft). Installed and enabled by deploy/harden.sh.\n# Emergency off (from a keyboard and screen on the Pi):  sudo systemctl disable --now surveillance-firewall\n[Unit]\nDescription=Surveillance camera firewall (only the tailnet reaches SSH and the web page)\nDefaultDependencies=no\nBefore=network-pre.target shutdown.target\nWants=network-pre.target\nConflicts=shutdown.target\n\n[Service]\nType=oneshot\nRemainAfterExit=yes\nExecStart=/usr/sbin/nft -f /opt/surveillance/deploy/firewall.nft\nExecStop=/usr/sbin/nft delete table inet cctv_filter\n\n[Install]\nWantedBy=sysinit.target\n'),
    ('deploy/sshd-hardening.conf', 0o644,
     '# Installed as /etc/ssh/sshd_config.d/10-surveillance.conf by deploy/harden.sh (Phase 14).\n# The name starts with 10- so it is read before other drop-ins such as 50-cloud-init.conf:\n# for sshd the first value it finds for a setting wins.\n\n# Only keys; no passwords, no keyboard-interactive, no root, only your account.\nPubkeyAuthentication yes\nAuthenticationMethods publickey\nPasswordAuthentication no\nKbdInteractiveAuthentication no\nPermitEmptyPasswords no\nPermitRootLogin no\nAllowUsers ADMIN_USER\n\nMaxAuthTries 3\nMaxSessions 4\nLoginGraceTime 30\nClientAliveInterval 300\nClientAliveCountMax 2\n\n# `ssh -L` (the tunnel to the web page) keeps working; everything else is switched off.\nAllowTcpForwarding local\nAllowStreamLocalForwarding no\nAllowAgentForwarding no\nX11Forwarding no\nPermitTunnel no\nGatewayPorts no\n'),
    ('deploy/apt-auto-upgrades.conf', 0o644,
     '// Installed as /etc/apt/apt.conf.d/20auto-upgrades by deploy/harden.sh (Phase 14).\n// Every day: refresh the package lists and install Debian stable and security updates (the\n// unattended-upgrades default selection). Raspberry Pi packages (kernel, firmware, libcamera,\n// picamera2) are left for a monthly manual update, so a change there never surprises a camera\n// nobody is watching.\nAPT::Periodic::Update-Package-Lists "1";\nAPT::Periodic::Unattended-Upgrade "1";\nAPT::Periodic::AutocleanInterval "7";\n'),
    ('tools/phase14_security_audit.py', 0o644,
     '#!/usr/bin/env python3\n"""Phase 14: read-only security audit of the Pi (run as root, it reads system files).\n\n    sudo python3 /opt/surveillance/tools/phase14_security_audit.py\n\nChecks what is really in effect, not what the configuration files say: the SSH server\'s\neffective settings and keys, the firewall, which ports listen and whether anything outside\nthe tailnet can reach them, sudo rules, automatic updates, the service sandboxes, file\npermissions and login accounts. Changes nothing.\n"""\nfrom __future__ import annotations\n\nimport json\nimport os\nimport pwd\nimport re\nimport stat\nimport subprocess\nimport sys\nfrom pathlib import Path\n\nAPP = Path("/opt/surveillance")\nDATA_DIRS = (Path("/etc/surveillance"), Path("/var/lib/surveillance"), Path("/var/log/surveillance"))\nSERVICES = ("surveillance-recorder", "surveillance-web", "surveillance-health")\nFIREWALL = "surveillance-firewall"\nTAILNET_ONLY_PORTS = {22, 443}\nMAX_EXPOSURE = 2.0\n\nresults: list[str] = []\n\n\ndef report(status: str, text: str) -> None:\n    results.append(status)\n    print(f"  [{status}] {text}")\n\n\ndef run(args: list[str]) -> subprocess.CompletedProcess:\n    try:\n        return subprocess.run(args, capture_output=True, text=True, timeout=60, check=False)\n    except (OSError, subprocess.TimeoutExpired) as exc:\n        return subprocess.CompletedProcess(args, 127, "", str(exc))\n\n\ndef check_ssh() -> None:\n    print("SSH server")\n    out = run(["sshd", "-T"])\n    if out.returncode != 0:\n        report("FAIL", f"cannot read the SSH server settings: {out.stderr.strip()[:200]}")\n        return\n    cfg: dict[str, str] = {}\n    for line in out.stdout.splitlines():\n        key, _, value = line.partition(" ")\n        cfg.setdefault(key, value)\n    expected = {"passwordauthentication": "no", "kbdinteractiveauthentication": "no", "permitrootlogin": "no",\n                "pubkeyauthentication": "yes", "permitemptypasswords": "no", "x11forwarding": "no",\n                "authenticationmethods": "publickey"}\n    for key, want in expected.items():\n        got = cfg.get(key, "?")\n        report("PASS" if got == want else "FAIL", f"{key} {got}" + ("" if got == want else f" (should be {want})"))\n    forwarding = cfg.get("allowtcpforwarding", "?")\n    report("PASS" if forwarding in ("local", "no") else "WARN", f"allowtcpforwarding {forwarding}")\n    tries = int(cfg.get("maxauthtries", "6") or 6)\n    report("PASS" if tries <= 3 else "WARN", f"maxauthtries {tries}")\n    users = cfg.get("allowusers", "").split()\n    report("PASS" if users else "WARN", f"allowusers {\' \'.join(users) or \'(anyone)\'}")\n    for name in users:\n        try:\n            home = Path(pwd.getpwnam(name).pw_dir)\n        except KeyError:\n            continue\n        keys_file = home / ".ssh" / "authorized_keys"\n        try:\n            keys = [line for line in keys_file.read_text().splitlines() if re.match(r"^(ssh-|ecdsa-|sk-)", line)]\n        except OSError:\n            keys = []\n        report("PASS" if keys else "FAIL", f"{name}: {len(keys)} key(s) in {keys_file}")\n        for path, limit in ((home, 0o755), (home / ".ssh", 0o700), (keys_file, 0o600)):\n            try:\n                mode = stat.S_IMODE(path.stat().st_mode)\n            except OSError:\n                continue\n            report("PASS" if mode & ~limit == 0 else "FAIL", f"{path} permissions {mode:o}")\n    prefs = run(["tailscale", "debug", "prefs"])\n    if prefs.returncode == 0:\n        try:\n            tailscale_ssh = json.loads(prefs.stdout).get("RunSSH", False)\n            report("PASS" if not tailscale_ssh else "WARN",\n                   "Tailscale SSH off (OpenSSH with keys is used)" if not tailscale_ssh else "Tailscale SSH is on")\n        except ValueError:\n            pass\n\n\ndef check_firewall() -> set[int]:\n    print("Firewall")\n    active = run(["systemctl", "is-active", FIREWALL]).stdout.strip() == "active"\n    enabled = run(["systemctl", "is-enabled", FIREWALL]).stdout.strip() == "enabled"\n    report("PASS" if active and enabled else "FAIL",\n           f"{FIREWALL}: {\'active\' if active else \'NOT active\'}, {\'on at boot\' if enabled else \'NOT on at boot\'}")\n    chain = run(["nft", "list", "chain", "inet", "cctv_filter", "input"]).stdout\n    report("PASS" if "policy drop" in chain else "FAIL", "incoming connections are dropped unless allowed"\n           if "policy drop" in chain else "the firewall table is not loaded")\n    if run(["systemctl", "is-enabled", "nftables"]).stdout.strip() == "enabled":\n        report("WARN", "nftables.service is enabled; its \'flush ruleset\' would also remove Tailscale\'s rules")\n    return TAILNET_ONLY_PORTS if active and "policy drop" in chain else set()\n\n\ndef check_ports(firewall_ports: set[int]) -> None:\n    print("Listening ports (what the network could reach)")\n    out = run(["ss", "-Hlntup"]).stdout\n    for line in sorted(set(out.splitlines())):\n        parts = line.split()\n        if len(parts) < 5:\n            continue\n        proto, local = parts[0], parts[4]\n        addr, _, port_text = local.rpartition(":")\n        addr = addr.strip("[]").split("%")[0]\n        if not port_text.isdigit() or addr in ("127.0.0.1", "::1") or addr.startswith("127."):\n            continue\n        port = int(port_text)\n        name = line.split(\'users:(("\', 1)[1].split(\'"\', 1)[0] if \'users:(("\' in line else "?"\n        where = f"{proto} {addr or \'*\'}:{port} ({name})"\n        tailnet_addr = addr.startswith("100.") or addr.startswith("fd7a:115c:a1e0")\n        if proto == "udp" and name == "tailscaled":\n            report("PASS", f"{where}: Tailscale\'s encrypted traffic")\n        elif not firewall_ports:\n            report("WARN", f"{where}: reachable from the home network (no firewall)")\n        elif proto == "tcp" and port in firewall_ports:\n            report("PASS", f"{where}: only through the tailnet (firewall)")\n        else:\n            report("PASS", f"{where}: blocked by the firewall" + (" (tailnet address)" if tailnet_addr else ""))\n\n\ndef check_sudo() -> None:\n    print("sudo")\n    files = [Path("/etc/sudoers"), *sorted(Path("/etc/sudoers.d").glob("*"))]\n    nopasswd = []\n    for path in files:\n        try:\n            lines = path.read_text().splitlines()\n        except OSError:\n            continue\n        nopasswd += [f"{path}: {line.strip()}" for line in lines if "NOPASSWD" in line and not line.lstrip().startswith("#")]\n    if nopasswd:\n        for entry in nopasswd:\n            report("WARN", f"sudo without a password: {entry}")\n    else:\n        report("PASS", "sudo always asks for the password")\n    members = run(["getent", "group", "sudo"]).stdout.strip().split(":")[-1]\n    report("PASS", f"members of the sudo group: {members or \'none\'}")\n\n\ndef check_updates() -> None:\n    print("Updates")\n    installed = run(["dpkg-query", "-W", "-f=${Status}", "unattended-upgrades"]).stdout.endswith("installed")\n    config = run(["apt-config", "dump"]).stdout\n    enabled = \'APT::Periodic::Unattended-Upgrade "1";\' in config\n    report("PASS" if installed and enabled else "FAIL",\n           "automatic security updates on" if installed and enabled else "automatic security updates are off")\n    timer = run(["systemctl", "is-enabled", "apt-daily-upgrade.timer"]).stdout.strip()\n    report("PASS" if timer == "enabled" else "WARN", f"apt-daily-upgrade.timer {timer}")\n    log = Path("/var/log/unattended-upgrades/unattended-upgrades.log")\n    if log.exists():\n        last = next((line for line in reversed(log.read_text(errors="replace").splitlines()) if line.strip()), "")\n        print(f"         last update run: {last[:110]}")\n    if Path("/run/reboot-required").exists():\n        report("WARN", "an update needs a reboot (Settings page, Restart Pi)")\n\n\ndef check_services() -> None:\n    print("Camera services")\n    for unit in SERVICES:\n        state = run(["systemctl", "is-active", unit]).stdout.strip()\n        out = run(["systemd-analyze", "security", f"{unit}.service", "--no-pager"]).stdout\n        match = re.search(r"Overall exposure level for \\S+: ([\\d.]+)", out)\n        exposure = float(match.group(1)) if match else 10.0\n        ok = state == "active" and exposure <= MAX_EXPOSURE\n        report("PASS" if ok else "WARN", f"{unit}: {state}, sandbox exposure {exposure:.1f} (0 best, 10 none)")\n    rule = Path("/etc/polkit-1/rules.d/50-surveillance.rules")\n    if rule.exists():\n        st = rule.stat()\n        report("PASS" if st.st_uid == 0 and stat.S_IMODE(st.st_mode) & 0o022 == 0 else "FAIL",\n               f"{rule}: owned by root, not writable by others")\n\n\ndef check_files() -> None:\n    print("File permissions")\n    writable = [p for p in [APP, *APP.rglob("*")] if p.lstat().st_uid != 0 or p.lstat().st_mode & 0o022]\n    report("PASS" if not writable else "FAIL", f"{APP}: owned by root and read-only to the services"\n           + (f"; {len(writable)} exception(s), e.g. {writable[0]}" if writable else ""))\n    for directory in DATA_DIRS:\n        if not directory.exists():\n            continue\n        others = [p for p in [directory, *directory.rglob("*")] if p.lstat().st_mode & 0o007]\n        report("PASS" if not others else "FAIL", f"{directory}: not readable by other users"\n               + (f"; {len(others)} exception(s), e.g. {others[0]}" if others else ""))\n    for path in (Path("/usr/local/sbin/cctv-tool"), *Path("/etc/systemd/system").glob("surveillance-*.service")):\n        if path.exists():\n            st = path.lstat()\n            report("PASS" if st.st_uid == 0 and st.st_mode & 0o022 == 0 else "FAIL", f"{path}: root-owned, not writable by others")\n\n\ndef check_accounts() -> None:\n    print("Accounts")\n    logins = [u for u in pwd.getpwall() if u.pw_shell not in ("/usr/sbin/nologin", "/sbin/nologin", "/bin/false", "")\n              and (u.pw_uid >= 1000 or u.pw_uid == 0) and u.pw_name != "nobody"]\n    report("PASS", f"accounts that can log in: {\', \'.join(u.pw_name for u in logins)}")\n    status = run(["passwd", "-S", "root"]).stdout.split()\n    locked = len(status) > 1 and status[1] in ("L", "LK", "NP")\n    report("PASS" if locked else "WARN", "root has no usable password" if locked else "root has a password set")\n    try:\n        cctv = pwd.getpwnam("cctv")\n        report("PASS" if cctv.pw_shell.endswith("nologin") else "FAIL", f"service user cctv: shell {cctv.pw_shell}")\n    except KeyError:\n        report("WARN", "service user cctv does not exist (Phase 13 not installed?)")\n\n\ndef main() -> int:\n    if os.geteuid() != 0:\n        print("Run it with sudo: sudo python3 /opt/surveillance/tools/phase14_security_audit.py", file=sys.stderr)\n        return 2\n    check_ssh()\n    firewall_ports = check_firewall()\n    check_ports(firewall_ports)\n    check_sudo()\n    check_updates()\n    check_services()\n    check_files()\n    check_accounts()\n    fails, warns = results.count("FAIL"), results.count("WARN")\n    verdict = "PROBLEMS FOUND" if fails else ("OK WITH WARNINGS" if warns else "HARDENED")\n    print(f"\\nRESULT: {verdict}  ({results.count(\'PASS\')} pass, {warns} warn, {fails} fail)")\n    return 1 if fails else 0\n\n\nif __name__ == "__main__":\n    sys.exit(main())\n'),
]

texts = {}
for rel, old, new in EDITS:
    text = texts.setdefault(rel, Path(rel).read_text())
    if text.count(old) != 1:
        sys.exit(f"ABORTED, nothing changed: {rel} does not match the expected 0.13.2 code "
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
print("Update to 0.14.0 complete.")
PYEOF
python3 update_to_0_14_0.py
sha256sum app/__init__.py tools/phase12_tailscale_check.py deploy/harden.sh deploy/firewall.nft deploy/surveillance-firewall.service deploy/sshd-hardening.conf deploy/apt-auto-upgrades.conf tools/phase14_security_audit.py
sudo bash ~/surveillance/deploy/install.sh
```

Expected checksums:

```
f44da4762ba8484e23b8e381d5efeeb018247aa8d05cfe8b0cabfbba66c28c6e  app/__init__.py
3496a0288c4c7bea3f9057c0ac10e5cb77f6b28d7746bb390dfde589fb7143c9  tools/phase12_tailscale_check.py
81e3f87b30633032b335b12c2ac12e067b246d697318239b8cfdc03629abfa75  deploy/harden.sh
557f1b22f6fcd4443ee3267221446090923206ae418935cd6cf32fe9f73402f1  deploy/firewall.nft
1dce7cdbadf58265098804c798fa93edba9ccaeea7f66c633a0d7560d4608086  deploy/surveillance-firewall.service
77759b2e214047c1aa25799e8cb7e36de08fc372dda80130faa82dc855f7bf32  deploy/sshd-hardening.conf
2decc65d5b15f3942f471f154b365bfbac0d389951e90c0fef6b56029672a3e1  deploy/apt-auto-upgrades.conf
ba2cf4fd9a42866fdf401dc8c54cd0a981c4115c694d89140f35c9bd55829567  tools/phase14_security_audit.py
```

The installer should end with `version 0.14.0` and three services each `active, 0 restart(s)`.

## Step 2 — Create an SSH key on your Windows laptop

Open **PowerShell** on the laptop:

```powershell
ssh-keygen -t ed25519 -C "laptop-cam01"
```

- Press Enter to accept the default file location. If it asks to overwrite an existing key, answer `n` and just use the one you have.
- Choose a **passphrase**. It protects the key if the laptop is stolen.

Copy the public key to the Pi over the tailnet. This asks for the Pi password one last time. The first connection to the name `cam01` also asks you to accept its fingerprint: type `yes`.

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh ysak@cam01 "umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys"
ssh ysak@cam01
```

The second command should ask for your **key passphrase** (`Enter passphrase for key ...`), not `ysak@cam01's password`. Stay logged in to this session for the next steps.

A second key (from another computer, or a backup copy of this one) guards against losing the laptop. You can append more keys the same way.

## Step 3 — SSH: keys only

In the key session from Step 2:

```bash
sudo bash ~/surveillance/deploy/harden.sh ssh
```

Expected output:

```
a key login for ysak was seen
sshd reloaded (your current session stays open)
  ...
  passwordauthentication no
  permitrootlogin no
  allowusers ysak
  authenticationmethods publickey
```

Keep that session open, and in a **new** PowerShell window check both of these:

```powershell
ssh ysak@cam01
ssh -o PubkeyAuthentication=no ysak@cam01
```

The first should log you in with your key. The second should print `ysak@cam01: Permission denied (publickey).`

If the first one fails, undo it from the session you kept open: `sudo rm /etc/ssh/sshd_config.d/10-surveillance.conf && sudo systemctl reload ssh`.

## Step 4 — Firewall

```bash
sudo bash ~/surveillance/deploy/harden.sh firewall
```

It prints the rules and `It switches itself off at HH:MM:SS unless you confirm.` Within those 3 minutes, open a **new** PowerShell window and run:

```powershell
ssh ysak@cam01
```

Then, in that new session:

```bash
sudo bash ~/surveillance/deploy/harden.sh firewall-confirm
```

Expected: `Firewall kept, and on at every boot`.

Now check that the home network is shut out. On the Pi, run `hostname -I` and note the first address (for example `192.168.0.50`). Then in PowerShell on the laptop:

```powershell
Test-NetConnection 192.168.0.50 -Port 22
```

Expected: `TcpTestSucceeded : False` after a few seconds. Also check that `https://cam01.tail1c1671.ts.net` still works from the laptop and from the phone on mobile data.

From now on, always connect with `ssh ysak@cam01`. The old `ysak.local` name no longer resolves, because the firewall blocks that lookup too, and SSH over the home network is blocked anyway.

## Step 5 — Automatic updates

```bash
sudo bash ~/surveillance/deploy/harden.sh updates
```

Expected:
- `APT::Periodic::Unattended-Upgrade "1";`
- The origin patterns: `label=Debian` and `label=Debian-Security`.
- Two timers with their next run times.
- A trial-run summary.

Once a month, update the Raspberry Pi packages by hand:

```bash
sudo apt update && sudo apt full-upgrade
sudo reboot
```

After the reboot, check the dashboard.

## Step 6 — Audit

```bash
sudo python3 /opt/surveillance/tools/phase14_security_audit.py
sudo cctv-tool phase12_tailscale_check
```

Expected: the audit ends with `RESULT: HARDENED` or `OK WITH WARNINGS`, and 0 fail. The Tailscale check should now report SSH as `the firewall blocks it from the home network (open inside the tailnet)`, with no warnings.

- If the audit warns `sudo without a password: /etc/sudoers.d/010_pi-nopasswd`, send it to me. Raspberry Pi OS sometimes installs that file, and it should go.
- Any other WARN or FAIL, send the line as it is.

**If you ever lock yourself out:** SSH changes don't affect logging in locally. Connect a keyboard and screen to the Pi, log in with your password, and run either of these:

```bash
sudo systemctl disable --now surveillance-firewall
sudo rm /etc/ssh/sshd_config.d/10-surveillance.conf && sudo systemctl reload ssh
```

**What's still true (not 100% secure):**
- Anyone with your laptop *and* your key passphrase can log in over SSH.
- Anyone with your Tailscale account and an approved device can reach the login page, but still needs the camera password.
- The recordings on the SD card aren't encrypted, so whoever takes the Pi itself has the footage. Mounting it somewhere hard to reach helps.

Please send me:
1. The checksums.
2. The results of Steps 3 to 5.
3. The full audit output from Step 6.

Then comes Phase 15, the stress tests:
- a power cut while recording;
- a full disk;
- unplugging the camera;
- losing the network;
- a 72-hour soak test with CPU and temperature logging.
