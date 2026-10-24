# AGENTS.md — running this repo in the Base44 sandbox

Non-obvious setup facts for the `flipperone-testing` monorepo. Read the README
and `fake-flipctl2/fake_flipctl2_CLAUDE.md` for what the app itself does.

## Bring-up

```bash
docker compose -f docker-compose.base44.yml up -d --build
```

- **No dependencies, no build step.** There is no `package.json` anywhere: the
  two servers are plain Node (`var`/`require`, Node 22 is enough) and the client
  is vanilla JS loaded by `<script>` tags. Never add an install step.
- The app is served from the bind-mounted checkout (`./:/app`), so a source edit
  is visible on reload with no rebuild.
- `web` runs `fake-flipctl2/server.js` (internal :8899 → host **3000**);
  `ui-sandbox` runs `fake-flipctl2/ui-sandbox.js` (internal/host **8900**).
- Root is required: `server.js` exits with "must be run as root on Linux" when
  `process.getuid() !== 0`. The `node:22` image runs as root, which satisfies it.

## Sandbox-only override: preview mode

`js/main.js` is written for the device and for a **PC viewer watching the
device**: a non-COG browser normally enters "mirror mode" — it shows the frame
the device posts to `/api/screen` and forwards keys to the device, and never
runs its own UI session. In the sandbox there is no device, so that path would
render an empty panel.

- In preview mode the page runs the **local scene session instead** (the COG
  path), keeping the existing desktop framing (Flipper One photo + click map).
  The click-map pads drive the local `Input` instead of the device mirror.
- The switch is gated on an **exact** `process.env.BASE44_PREVIEW_MODE === '1'`
  check in `server.js` (`PREVIEW_MODE`), which injects
  `window.__BASE44_PREVIEW_MODE__ = true` into the served `index.html`, plus
  `IS_PREVIEW` in `js/main.js`. Compose passes the variable through as a bare
  `BASE44_PREVIEW_MODE` entry — never hardcode it to `1`.
- **With the variable unset or any other value** nothing is injected: the
  device (COG UA) and real PC viewers behave exactly as before (mirror mode).
- Verify it: `curl -fsS http://localhost:3000/ | grep __BASE44_PREVIEW_MODE__`
  (present only in preview mode), and check that the canvas renders the UI and
  responds to keyboard input / the on-photo pads.

## Sandbox-only limitations (expected, not bugs)

`server.js` is written for the Flipper One device. Every device integration is
best-effort (`tryExec` swallows failures and caches "unavailable"), so the UI
still renders and navigates in the sandbox — but these do not work here:

- `mmcli`/`qmicli` 5G modem, `nmcli`/`iw` Wi-Fi + ethernet, `ip` routing, `df`
  eMMC disk, sysfs battery (`/sys/class/power_supply/bq28z610-0`) and thermal
  zones, `arecord`/`sox` audio, `cec-ctl` TV control, ffmpeg frame sheets.
- `/media/ui_png` and `/flipperone-testing/sound/audio_files` do not exist, so
  the UI-PNG viewer and the sound picker list nothing.
- **Variant switch does not work here.** `POST /api/switch/flipctl` and
  `POST /api/system/reboot` shell out to `systemd-run`/`systemctl`, which exist
  on neither the sandbox nor a container. They answer `200` and then do nothing,
  and `fake-flipctl` (the other variant, sibling directory) is consequently never
  served — only `fake-flipctl2` runs. The `active-flipctl` symlink in this repo
  points at `/flipperone-testing/fake-flipctl2`, an absolute device path that does
  not resolve in the sandbox; nothing in the runtime path depends on it.
- OTA update (`/api/update/check|apply`) runs `git fetch` against
  `/flipperone-testing`, which is not a git checkout here.

## Verifying it works

1. `curl -fsS http://localhost:3000/api/version` → `{"id":"<hex>"}` (a fresh id
   per server start; the client polls it and reloads the page when it changes).
2. Load `http://localhost:3000/` — a 256×144 canvas rendering the Flipper One UI.
   Keyboard only: arrows navigate, Enter = ok, Escape = back (`js/input.js` maps
   `i/j/k/l/m/n` as well).
3. `curl -fsS http://localhost:3000/api/hostname` → container hostname; the
   device-data endpoints (`/api/power`, `/api/modem`, `/api/ethernet`, …) return
   `{"available":false}` / empty payloads as described above.
4. UI sandbox: load `http://localhost:8900/` (component library grid).
