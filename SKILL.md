---
name: phone-lane
description: Reactive ADB phone lane for the Pixel 9 Pro XL via the awrawr-pc bridge.
---

# phone-lane

Reactive ADB phone lane for the Pixel 9 Pro XL (Android 17) via awrawr-pc.
The phone is reached over LAN wireless debugging (`10.0.0.77`, port rotates);
the lane host is **awrawr-pc** (`/usr/bin/adb`, `~/.local/bin/mobilerun`).
The cell cannot see the phone directly -- all device work goes through the
bridge (`~/workspace/skills/awrawr-mcp/bin/exec.py`).

## Binaries (run on awrawr-pc, live in `~/phone-lane/`)

- `bin/phone-lane-keeper [--once]` — self-healing ADB daemon. Watches
  `adb devices`; on Pixel drop it rediscovers the wireless-debugging port
  with a sub-second parallel scan (20k ports, ~1s), reconnects, verifies
  with `mobilerun ping`, pins the target to `~/phone-lane/pinned_target`,
  and appends transition rows to `~/phone-lane/ledger.jsonl`. Never touches
  other devices (SmartTV at 10.0.0.225:5555, emulator-5554). Hardened
  candidate ordering (2026-09-19): pinned target first, then ascending scan
  results; `44933` (dead wireless-debug endpoint) is explicitly rejected
  even if open; every candidate must prove identity (`adb -l` model
  `Pixel_9_Pro_XL` on the exact serial AND `getprop ro.product.model ==
  "Pixel 9 Pro XL"`) before pinning -- failures are `adb disconnect`ed and
  logged as `candidate_rejected`. Runs as a systemd user service
  (`phone-lane-keeper.service`), reboot-persistent via `linger`.
- `bin/phone-vitals [--once] [--saver]` — live telemetry publisher. Every
  60s publishes battery/screen/focused-app/thermal/network to
  `~/phone-lane/vitals.jsonl` + `vitals-latest.json`. Reactive: battery
  <= 20% and thermal throttling raise alert rows; `--saver` also engages
  battery saver on critical battery. Runs as a systemd user service
  (`phone-vitals.service`), reboot-persistent via `linger`.
- `bin/morphe-tax-collector [--init] [--once]` — the Morphe tax verifier.
  Checks installed Aura (`com.facebook.aura`) versionName/versionCode /
  lastUpdateTime / installer, sha256 of the installed APK vs
  `~/phone-lane/aura_baseline.json` (patch-drift detection), and probes
  the app data dir via `run-as` for login preservation (filenames/sizes
  only, never contents; reports `unknown_no_root` honestly when it cannot
  verify). Appends verdict rows to `~/phone-lane/morphe-tax.jsonl`.

## Standing facts

- Phone: Pixel 9 Pro XL (`komodo`), Android 17, Mobilerun Portal 0.7.25,
  Aura 7.0.0.25.163 (Morphe-patched, sideloaded: `installerPackageName=null`).
- Chris's order: the phone's local-storage login must NEVER be removed --
  the tax collector guards this invariant.
- Wireless-debugging port rotates (seen: 41043, 42831, 43585, 44933,
  36477...). Never hardcode a port; always read `pinned_target` or heal.
- Pairing persists in `~/.android/adbkey*` on awrawr-pc; no USB needed.

## Deploy

```bash
# from the cell:
~/workspace/skills/awrawr-mcp/bin/xfer.py put \
  ~/workspace/skills/phone-lane/bin/phone-lane-keeper /home/toxic/phone-lane/bin/phone-lane-keeper
# ... same for the other two, then on awrawr-pc:
chmod +x ~/phone-lane/bin/*
setsid nohup ~/phone-lane/bin/phone-lane-keeper >> ~/phone-lane/keeper.log 2>&1 < /dev/null &
setsid nohup ~/phone-lane/bin/phone-vitals >> ~/phone-lane/vitals.log 2>&1 < /dev/null &
```

## Deploy (first-class, 2026-09-19)

Keeper and vitals run as **systemd user services** on awrawr-pc --
`phone-lane-keeper.service` / `phone-vitals.service` in
`~/.config/systemd/user/`, `Restart=always`, enabled under `default.target`.
`loginctl enable-linger toxic` is on, so they survive reboot. Old
`setsid nohup` daemons were replaced; do not reintroduce them.

```bash
# from the cell: xfer the binaries, then on awrawr-pc:
systemctl --user daemon-reload
systemctl --user enable --now phone-lane-keeper.service phone-vitals.service
systemctl --user is-active phone-lane-keeper.service phone-vitals.service
```

Kill-test receipt 2026-09-19T09:26:25Z: `adb disconnect 10.0.0.77:36477` ->
`lane_down_detected` (09:26:18) -> `heal_scan` (candidates
`[10.0.0.77:36477, 10.0.0.77:44787]`, pinned first, scan 975ms) ->
`healed` target `10.0.0.77:36477`, `mobilerun_ping_ok=true`, `heal_ms=6493`.
Identity proof: `adb -l` model `Pixel_9_Pro_XL` + `getprop ro.product.model`.
SmartTV untouched. Aura login preservation remains unproven
(`unknown_no_root`) -- never claimed otherwise.
