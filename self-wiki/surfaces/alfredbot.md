# AlfredBot — Physical Robot Companion

AlfredBot is a physical robot running on a Raspberry Pi with a 7-inch display
and a Chromium kiosk. It is not a Discord bot.

Entry point: `apps/alfredbot/src/host/main.ts` — a Hono HTTP server on port
3200.

## Architecture

```
Mac (Conversation Core + GPT-Live) ←── LiveKit WebRTC ──→ Robot (kiosk participant)
                                 ←── HTTP status polling ──→ Robot host (port 3200)
                                 ←── show_expression events ─→ Robot face/neck
```

The robot is a pairing + LiveKit participant, same shape as the iOS client.
It does not run its own LLM, memory, or session logic — those stay on the Mac
via `pnpm desktop` + `make alfred VOICE=live`.

## On-disk layout on the Pi

- `/opt/alfredbot/` — host binary + UI
- `/etc/alfredbot.env` — environment
- `alfredbot.service` — Node host on `0.0.0.0:3200`
- `alfredbot-kiosk.service` — Chromium kiosk on the 7" panel
- `data/alfredbot/identity.json` — device identity store
- `/tmp/alfred-head-stick.json` — joystick state for neck servos
- `/tmp/alfred-tracker.json` — face tracker telemetry
- `/tmp/alfred-cli-mode.json` / `/tmp/alfred-tracker-pause.json` — tracker control

## Startup flow

Four screens in a boot sequence, managed by `screenFor()` in `routes.ts`:

1. **Wi-Fi** — Scan and join a network via `nmcli`
2. **Claim** — Show QR code; the iPhone scans it and links via `alfrd.net` cloud
3. **PIN** — Device pairing: request device_id, then confirm the 6-digit PIN
   shown on the Mac
4. **Face** — Ready state; conversation and expression events drive the display

## Hardware

The Pi controls physical hardware through Python helper scripts:

- **Neck** — PCA9685 servo board (I2C channels: body/0, neck/1, tilt/2, roll/3)
  with joystick control (deadzone 0.08, max degrees/sec per axis)
- **Face tracking** — Python `alfred-face-tracker.service`, toggled on/off via
  systemctl commands
- **Wheels** — Two-channel motor control, stick interface with -1..1 range
- **Arm** — Servo arm with a wave animation sequence
- **Camera** — Raspberry Pi CSI camera via `rpicam-vid`, off by default
  (`ALFREDBOT_ENABLE_RPI_CAMERA=1` to enable); JPEG frame sink at
  `/tmp/alfred-camera-latest.jpg` for the face tracker

All hardware paths have mock fallbacks when `process.platform === "darwin"`.

## Conversation integration

The robot watches the Mac's conversation state by polling
`/api/session/status` every 400ms:

```ts
const res = await desktopFetch(store.serverUrl, store.cloudToken,
  store.deviceToken, "/api/session/status");
```

`conversationSnapshot()` reports whether either iPhone Talk or the desktop
backend is currently active. The robot uses this to drive `show_expression`
face events (smile, frown, wink, etc.) from the GPT-Live voice model.

Talk control: `/api/talk/start` mints a LiveKit token from the desktop,
`/api/talk/stop` ends the session, `/api/talk/listen`/`/api/talk/hangup` toggle
the robot's audio state. Audio source can switch between phone and local robot
microphone via `/api/audio/source`.

## Deployment from the Mac

Build on the Mac, rsync a payload to the Pi:

```bash
cd apps/alfredbot
./deploy/push.sh alfred@100.115.52.14
ssh alfred@100.115.52.14 '~/alfredbot/deploy/bootstrap-on-pi.sh'
```

`bootstrap-on-pi.sh` writes the env and systemd units, then switches boot so the
new host + kiosk start instead of any old Electron kiosk. The old app stays on
disk (disabled, not uninstalled).

## Environment

| Variable | Effect | Default |
|---|---|---|
| `ALFREDBOT_PORT` | Host port | `3200` |
| `ALFREDBOT_CLOUD_URL` | Control plane URL | `https://api.alfrd.net` |
| `ALFREDBOT_STORE_PATH` | Identity file path | `data/alfredbot/identity.json` |
| `ALFREDBOT_SKIP_WIFI` | Skip Wi-Fi screen | (unset) |
| `ALFREDBOT_FORCE_WIFI_UI` | Always show Wi-Fi screen | (unset) |
| `ALFREDBOT_DEVICE_NAME` | Pairing display name | (unset) |
| `ALFREDBOT_BIND` | Listen address | `0.0.0.0` |
| `ALFREDBOT_ENABLE_RPI_CAMERA` | Enable CSI camera | (unset) |
| `ALFREDBOT_CAMERA_FRAME_SINK` | JPEG path for face tracker | `/tmp/alfred-camera-latest.jpg` |
| `ALFREDBOT_HEAD_MOCK` | Force mock head hardware | auto (darwin mock) |
| `ALFREDBOT_PYTHON` | Python binary for hardware control | auto-detect |
| `ALFREDBOT_HEAD_STICK` | Path to head-stick.py | auto-detect |
| `ALFREDBOT_HEAD_PREFS` | Path to head tracking prefs | `/var/lib/alfredbot/head-prefs.json` |

## API surface

Key routes on the host (full list in `routes.ts`):

| Route | Purpose |
|---|---|
| `GET /api/status` | Boot state: screen, paired, connection info |
| `GET /api/identity` | Device identity card |
| `POST /api/provision` | Link bot to cloud + desktop |
| `POST /api/provision-pair` | Complete PIN pairing |
| `POST /api/claim` | Scan claim QR payload, link via cloud |
| `POST /api/pair/request` | Request device_id from desktop |
| `POST /api/pair/confirm` | Confirm 6-digit PIN with desktop |
| `POST /api/unpair` | Clear store and unpair from desktop |
| `GET /api/talk/status` | Conversation status from desktop |
| `POST /api/talk/start` | Mint LiveKit token, start talk |
| `POST /api/talk/stop` | End session on desktop + locally |
| `POST /api/talk/listen` / `POST /api/talk/hangup` | Audio state |
| `POST /api/head/stick` / `POST /api/head/release` | Joystick neck control |
| `POST /api/head/tracking` | Enable/disable face tracking |
| `GET /api/head/status` | Head hardware status |
| `POST /api/wheels/stick` / `POST /api/wheels/release` | Drive control |
| `POST /api/arm/wave` | Wave animation |
| `POST /api/camera/start` / `POST /api/camera/preview` | RPi camera |
| `POST /api/wifi/scan` / `POST /api/wifi/join` | Wi-Fi management |

## Related

- [../capabilities/delegation.md](../capabilities/delegation.md) — the desktop
  routes this polls
- [voice-agent.md](voice-agent.md) — the voice stack the robot participates in
- [ios-client.md](ios-client.md) — parallel mobile surface, same pairing shape