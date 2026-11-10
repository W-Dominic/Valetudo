# Handoff: Dreame L10s Ultra running Valetudo

Last updated: 2026-11-09

## Status
**Rooted and running Valetudo, cloud-free, on the home network.**

| | |
|---|---|
| Robot | Dreame L10s Ultra (original, not Gen2), serial prefix `R2228`, model ID `dreame.vacuum.r2228o`, SoC Allwinner MR813 |
| Valetudo | `2026.10.0`, commit `78267517` (binary `aarch64`), implementation `DreameL10SUltraValetudoRobot` |
| IP | `192.168.1.170` (set a DHCP reservation if not done yet) |
| Web UI | `http://192.168.1.170` (LAN only; Valetudo blocks requests from public IPs) |
| SSH | `ssh -i ~/.ssh/dreame-robot root@192.168.1.170` (private key also in 1Password, Ed25519) |
| Rooting method | Fastboot (`valetudo-helper-felroot`), firmware from the Dustfab (fab.valetudo.cloud), "FEL package" output |

This fork's `master` is 5 commits ahead of `78267517`; none touch the L10s Ultra code path, so `master` matches what runs on the robot.

## Backups (stored in the user's cloud drive)
- `backup.tar`: `/mnt/private/` + `/mnt/misc/` (calibration and identity data, incl. the factory cloud key at `/mnt/private/ULI/factory/key.txt`). **Sensitive. Irreplaceable.**
- `samples.zip`: recon samples (`dustx100-102.bin`, ~1 GB), for disaster recovery.
- Dustfab rooting tarball (re-flash without rebuilding) and FEL recon package.

## Breakout board (as built)
Design: `Hypfer/valetudo-dreameadapter`, `pcbs/dreamebreakout`, 1.2 mm PCB, revision 2022.09.0.

Minimum parts for Fastboot: 2.00 mm 2x8 male header (robot-facing), 6x6 mm tactile button (Boot_Select), USB-A cable wires. No jumpers, no USB-A/micro-USB sockets, no female headers.

USB wiring goes into the **SoC Breakout header holes** (1.0 mm), not the 0.44 mm micro-USB holes:

| SoC header pin | Net | Wire |
|---|---|---|
| 1 (square) | VBUS (robot side, direct) | none, red taped off |
| 2 | D- | white |
| 3 | D+ | green |
| 4 | OTG ID | none |
| 5 | GND | black |
| 6-8 | TX, RX, BSel | none |

Facts verified from the KiCad files:
- Micro-USB footprint J4: pads 1 VBUS (square), 2 D-, 3 D+, 4 ID, 5 GND. Pad 1 is `USB_VBUS`, which reaches robot VBUS only through JP1.
- JP1 "Micro USB VBUS" is a 2-pin 2.00 mm through-hole header (not a solder jumper). Leave open.
- JP2 OTG ID jumper ties ID to GND. Must be **open** for Fastboot.
- The USB-A footprint's VBUS is the robot's VBUS with no jumper. Never connect it to a PC with an A-to-A cable.
- Button SW1 shorts Boot_SEL to SoC_GND.
- Robot-facing pads (component side, in order): BSel, RX, TX, GND, ID, D+, D-, VBUS.
- Plug in with "This side towards front/top of robot" facing correctly. Plugged in backwards, the robot boots normally (startup music, red/orange LEDs) instead of entering fastboot.

Lessons from the build:
- "full-speed" + "device not responding to setup address" in `dmesg` meant a bad D- joint (D+ detection worked, data didn't). A full rebuild on a fresh PCB fixed it.
- "firmware failed to leave LPS state" in `dmesg` is the laptop's Realtek Wi-Fi driver, unrelated.

## How rooting was done (for a future re-root)
Debian 13 Live (KDE) on a laptop, root shell, helper run from `/root` (FAT sticks mount noexec), outputs saved to a second USB stick.

1. Enter fastboot: robot off, hold PCB button, hold power 5 s, release power, keep PCB button 3 s more, plug USB into laptop.
2. Recon: `boot fsbl.bin payload.bin`, `getvar config`, `oem stage0/1/2` each followed by `get_samples`, `reboot`, zip samples.
3. Build on the Dustfab (FEL package, SSH public key from 1Password).
4. Root: `boot`, `getvar config`, `oem dust <check.txt>`, `oem prep`, `flash toc1`, `flash boot1/rootfs1`, `flash boot2/rootfs2` (dtbo only if present), `reboot`.
5. Install: join robot AP, SSH in, run `valetudo-helper-httpbridge`, back up `/mnt/private` + `/mnt/misc`, `wget` the binary to `/tmp`, move to `/data/valetudo`, copy `/misc/_root_postboot.sh.tpl` to `/data/_root_postboot.sh`, `chmod +x` both, reboot.

Source of truth: valetudo.cloud, Installation, Dreame, Fastboot.

## Privacy / network audit (2026-11-09)
Valetudo's own outbound traffic (from source):
- NTP to `valetudo.pool.ntp.org` every 8 h (`backend/lib/res/default_config.json`).
- GitHub releases API, only when "check for updates" is pressed (`webserver/UpdaterRouter.js`).
- MQTT only if configured. mDNS (5353) and SSDP (1900) stay on the LAN.
- The "AI Assistant" is a joke: ELIZA running in the browser (`frontend/src/valetudo/ValetudoAI.tsx`), no network.

On the robot:
- `/etc/hosts` maps ~80 vendor cloud hostnames (Dreame, Xiaomi miio/FDS, Aliyun OSS, miui tracking, 3irobotix, Roborock) to `127.0.0.1`/`::1`. `resolv.conf` uses `192.168.1.1`, so unlisted names still resolve.
- `netstat -tunap`: no internet connections. Vendor processes (`ava`, `mediad`, `miio_client`, `miio_agent`) only talk over loopback; `miio_client` talks to Valetudo's dummycloud at `127.0.0.1:8053`.
- conntrack: only outbound flows were NTP and an ICMP ping to `8.8.8.8` from `/etc/rc.d/wifi_monitor.sh` (connectivity check, no payload).
- Vendor telemetry (`/usr/bin/report_data.sh`, cron every 1-5 min) is handed to `ava` via `avacmd ... upload_bd` and ends at Valetudo's dummycloud, which logs unknown messages locally (`backend/lib/miio/Dummycloud.js`). The scripts' `nslookup` calls are neutralized in the rooted firmware.
- `iptables`: no rules (all ACCEPT). The hosts file is the only barrier.
- If the internet ping fails but the gateway answers, `wifi_monitor.sh` only logs an event; it restarts the Wi-Fi driver only when the gateway is unreachable. So a router-level WAN block is safe.

Camera:
- Obstacle photos (672x504 JPEG) show as icons on the live map; click to view. Stored on the robot.
- Live stream: Options, Camera Streaming. Off by default; enabling requires a deliberate manual step on the robot. Disabling deletes the streamer binary.

## Open items
- [ ] Router: block WAN for `192.168.1.170`, allow LAN only.
- [ ] Point Valetudo NTP at `192.168.1.1` (or a local server), or disable it, once WAN is blocked.
- [ ] DHCP reservation for the robot.
- [ ] Charge fully, then the first mapping run with all doors open; then split/rename segments.
- [ ] Settings pass: timezone, dock (auto-empty, mop wash/dry), carpet, consumables.
- [ ] Remote access later: Tailscale option B, `tailscale serve --bg http://192.168.1.170` on the home server (needs MagicDNS + HTTPS certs). Works with Valetudo's external-access block because requests arrive from the server's LAN IP.
- [ ] Optional: Home Assistant + Mosquitto (MQTT autodiscovery), "I Can't Believe It's Not Valetudo" for map images.
- [ ] Code walkthrough: next stops `DreameGen2ValetudoRobot.js`, `DreameValetudoRobot.js`, `MiioValetudoRobot.js`.

## Other notes
- Never connect the robot to the Dreame app.
- No official iOS app; use the web UI (Add to Home Screen). One map only.
- Mop solution: plain or distilled water (user prefers non-toxic).
- Keep the breakout board for recovery.
- Support: Valetudo Telegram group.

## Side threads (lower priority)
- Homelab k3s: start single-node on the existing server; keep Home Assistant on Docker.
- Old ThinkPad with dead battery as a server/k3s node: enable "Power On with AC Attach" in BIOS.
- Separate ESP32 garden sensor project (LILYGO TTGO LoRa32), no overlap.
