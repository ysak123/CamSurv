Phase 12 works: the checker passed everything that matters. The page loads from your laptop and phone over the tailnet with a real Let's Encrypt certificate, Funnel is off, port 8080 is local-only, and the page is unreachable with Tailscale off. (I'm reading "can't access the internet" as "can't open the camera page". If your phone actually lost all internet with Tailscale off, tell me.)

Three results need a follow-up, plus one small fix to the checker:

- **Key expiry WARN:** my note in the checker was wrong. Tailscale disables key expiry only when a device first logs in with a tag. Tagging it afterwards in the admin console, as we did, leaves expiry on, so the camera would drop off the tailnet on 2027-03-23. You need to turn expiry off by hand (Step 14 below). The update script below corrects the message.
- **Port 111:** that's `rpcbind`, a helper for NFS file sharing that Raspberry Pi OS runs by default. It answers anyone on your home network, and the camera doesn't need it, so turn it off (Step 13).
- **The ping didn't test the policy:** 192.168.0.26 is your laptop's home-network address, which doesn't go through Tailscale. That 100% loss was most likely the laptop's own firewall. The retest is in Step 15.

The two "tailnet only" ports (55890 and 40880) are almost certainly Tailscale's own peer service. The access policy blocks them anyway, because it only lets your devices reach 443 and 22.

The `top` output also shows no `tailscaled` line. At about 12% CPU for the recorder, Live View probably wasn't streaming at the time. Step 16 measures it properly.

## Update the checker

```bash
cd ~/surveillance
cat > update_phase12_checker.py <<'EOF'
#!/usr/bin/env python3
"""Update the surveillance project Phase 12 checker fixes (run from ~/surveillance)."""
import shutil, sys
from pathlib import Path

EDITS = [
    ('tools/phase12_tailscale_check.py',
     'CAMERA_TAG = "tag:camera"\nHTTPS_TIMEOUT_S = 60\n\nresults: list[str] = []\n',
     'CAMERA_TAG = "tag:camera"\nHTTPS_TIMEOUT_S = 60\nKNOWN_PORTS = {\n    22: "SSH, restricted to the tailnet in Phase 14",\n    111: "rpcbind (an NFS helper this camera does not need): disable it, Phase 12 step 13",\n}\n\nresults: list[str] = []\n'),
    ('tools/phase12_tailscale_check.py',
     '    expiry = me.get("KeyExpiry")\n    if expiry:\n        report("WARN", f"device key expires {expiry[:10]}: the camera would drop off the tailnet then "\n                       f"(tagging it disables expiry)")\n    else:\n        report("PASS", "device key does not expire")\n',
     '    expiry = me.get("KeyExpiry")\n    if expiry:\n        report("WARN", f"device key expires {expiry[:10]}: the camera would drop off the tailnet then; "\n                       f"admin console, Machines, ... menu of this device, Disable key expiry")\n    else:\n        report("PASS", "device key does not expire")\n'),
    ('tools/phase12_tailscale_check.py',
     '            print(f"         tailnet only : {addr}:{port}{who}")\n        else:\n            label = "SSH, restricted to the tailnet in Phase 14" if port == 22 else "check what this is"\n            report("WARN", f"reachable from the home network: {addr or \'*\'}:{port}{who}: {label}")\n\n',
     '            print(f"         tailnet only : {addr}:{port}{who}")\n        else:\n            label = KNOWN_PORTS.get(port, "check what this is")\n            report("WARN", f"reachable from the home network: {addr or \'*\'}:{port}{who}: {label}")\n\n'),
]

texts = {}
for rel, old, new in EDITS:
    text = texts.setdefault(rel, Path(rel).read_text())
    if text.count(old) != 1:
        sys.exit(f"ABORTED, nothing changed: {rel} does not match the expected Phase 12 checker "
                 f"(found {text.count(old)} matches for:\n{old})")
    texts[rel] = text.replace(old, new)
for rel, text in texts.items():
    shutil.copy2(rel, rel + ".bak")
    Path(rel).write_text(text)
    print(f"updated {rel}  (backup: {rel}.bak)")
print("Checker updated.")
EOF
python3 update_phase12_checker.py
sha256sum app/__init__.py app/auth.py app/config.py app/main.py app/status.py app/web.py app/web_auth.py app/web_main.py app/web_settings.py tools/set_setting.py web/static/app.js tools/phase12_tailscale_check.py
```

Your last message had the Step 7 command but not its output, so please paste this `sha256sum` output. All lines should match the list I gave you, except the last one. The checker's new checksum is `ffcbb36d4d77b10ec28d97b2507af593f2cad9846eb606faa4bc258a113db5fa`.

## Step 13 — Turn off rpcbind

```bash
systemctl status rpcbind --no-pager | head -3
sudo systemctl disable --now rpcbind.service rpcbind.socket
sudo systemctl mask rpcbind.service rpcbind.socket
sudo ss -ltnup | grep ':111 ' || echo "port 111 closed"
```

Expected: `port 111 closed`. This closes both TCP and UDP 111. Masking stops other packages from quietly starting it again, and it's reversible with `sudo systemctl unmask rpcbind.service rpcbind.socket` if you ever need NFS.

## Step 14 — Disable the camera's key expiry

In the admin console, open **Machines**, click **⋯** next to `cam01`, then choose **Disable key expiry**.

## Step 15 — Prove the camera can't reach your devices

**A. Let Tailscale check it.** In **Access controls**, replace the `"tests"` block with the one below. Change `YOUR-LOGIN@example.com` to your login in every place, then **Save**.

```json
  "tests": [
    {"src": "YOUR-LOGIN@example.com",
     "accept": ["tag:camera:443", "tag:camera:22"],
     "deny": ["tag:camera:80", "tag:camera:8080"]},
    // The camera must not be able to open anything on your own devices.
    {"src": "tag:camera",
     "deny": ["YOUR-LOGIN@example.com:22", "YOUR-LOGIN@example.com:443",
              "YOUR-LOGIN@example.com:445", "YOUR-LOGIN@example.com:3389"]},
    {"src": "tag:camera", "proto": "icmp", "deny": ["YOUR-LOGIN@example.com:0"]}
  ]
```

Tailscale refuses to save the policy if any of these tests fails. So "saved" means the policy blocks the camera from reaching your devices.

**B. Retry the ping using the tailnet address.** On the Pi:

```bash
tailscale status
ping -c 3 <your laptop's 100.x.x.x address from that list>
```

Expected: `100% packet loss`.

## Step 16 — Measure CPU with a live viewer over the tailnet

Open **Live View** on your phone over the tailnet and leave it running. Then on the Pi:

```bash
top -b -n 3 -d 5 -p "$(pgrep -d, -x tailscaled),$(pgrep -d, -f 'app.main|app.web_main')"
```

This shows only the recorder, the web interface and `tailscaled`.

## Step 17 — Re-run the checker

```bash
python3 ~/surveillance/tools/phase12_tailscale_check.py
```

Expected: `device key does not expire` is now PASS, no port 111 lines, and only the two SSH WARNs (`0.0.0.0:22` and `:::22`) are left. Phase 14 restricts SSH to the tailnet.

Please send me the checksums, the output of Steps 13 and 15B, whether the policy saved with the new tests, the Step 16 `top` lines, and the checker output. Then we'll start Phase 13: systemd services so everything starts at boot and restarts on failure.
