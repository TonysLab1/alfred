# Voice Client — Browser Frontend

The voice client at `apps/voice-client/` is a vanilla TypeScript browser
application that connects to Alfred's voice stack via LiveKit WebRTC. It also
doubles as the desktop client's voice SPA when embedded at `/voice/`.

Entry point: `apps/voice-client/src/main.ts`.

## Build

Built with Vite (not React — no `react` or `react-dom` in `package.json`). Two
build modes:

- `pnpm client` — standalone at `http://localhost:5173/` (base `/`)
- `pnpm client:build:desktop` — embedded in desktop client at `/voice/`
  (`VOICE_CLIENT_BASE=/voice/`)

The Vite dev server includes a custom plugin (`tokenApiPlugin`) that mints
LiveKit access tokens at `/api/token` using `LIVEKIT_API_KEY`,
`LIVEKIT_API_SECRET`, `LIVEKIT_URL`, and `LIVEKIT_ROOM` from the repo `.env`.

## Dependencies

- `livekit-client` — WebRTC audio/video room connection
- `@elevenlabs/client` — speech engine for TTS output

No React, no framework library. The UI is built with direct DOM manipulation.

## Features

From the main entry point (`main.ts`), the voice client assembles these
components:

- **BriefingHud** — daily briefing display with weather, news, reminders
- **CaptionHud** — live caption overlay from `alfred.caption` and
  `alfred.user` LiveKit data topics
- **LiveWaveform** — animated audio waveform on `<canvas>`
- **TranscriptThread** — scrollable conversation transcript
- **Composer** — text input for text-only talk
- **HistoryDrawer** — conversation history navigation
- **UserTranscriptHud** — user-only captions from `alfred.user` topic

## Shell bridge

`shell-bridge.ts` detects whether the voice client is embedded in a shell
context and can navigate shell-level routes or post state to the embedding
window.

## Speech engine integration

`speech-engine-session.ts` manages the ElevenLabs speech engine session with
methods for: start/stop session, submit user text, handle interruptions,
enforce hush/mute state, and relay output audio tracks.

## How it connects

1. Fetch a LiveKit token from the dev server's `/api/token`
2. Join the LiveKit room (same room the voice agent occupies)
3. Publish local audio track (microphone)
4. Subscribe to remote audio from the backend
5. Listen for caption data on LiveKit data topics
6. Render briefing content parsed from data channel payloads

## Status display

The `#status` element shows connection state. The `#link-dot` reflects
LiveKit room connectivity. `#meta` shows provider/stack info. `#level-tag`
shows audio level. Transport controls: connect, mute, break-in (shhh), and
layout toggle.

## Related

- [voice-agent.md](voice-agent.md) — the backend the client connects to
- [desktop-text.md](desktop-text.md) — the desktop server that can host this at
  /voice/
- [simulator.md](simulator.md) — the test harness alternative to a live browser