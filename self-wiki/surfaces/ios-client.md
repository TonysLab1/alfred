# iOS Client — React Native Mobile Application

The iOS client at `apps/iOS-client/` is an Expo SDK 54 React Native application
(React Native 0.81.5, React 19.1.0). It is NOT a Swift application.

Entry point: Expo Router (`expo-router` 6.0.14).

## Stack

- **Framework:** Expo SDK 54 Dev Client — not Expo Go. Runs native code via
  dev client, prebuilt with `expo prebuild --platform ios`.
- **UI:** NativeWind (Tailwind CSS 3.4.18 for React Native),
  `@shopify/flash-list` for performant lists, `react-native-reanimated` 4.3.0
  for animations, `react-native-skia` for 2D graphics.
- **Navigation:** `@react-navigation/native-stack` + bottom tabs +
  material top tabs + drawer — full navigation suite.
- **Voice:** LiveKit via `@livekit/react-native` (2.12.0), WebRTC, and
  `@livekit/react-native-callkeep` for iOS CallKit integration. `expo-audio`
  for playback.
- **Camera:** `react-native-vision-camera` 4.7.2 and Expo Camera/ImagePicker/
  ImageManipulator for photo capture and processing.
- **Services:** Nearly every Expo module: Calendar, Contacts, Location,
  Notifications, SecureStore, FileSystem, Sensors, Haptics, Speech,
  BackgroundFetch, Video, WebBrowser, SMS, MailComposer, Print, Battery,
  Network, Cellular, ScreenCapture, Brightness.
- **Cloud:** `alfrd.net` via `cloud-api.ts` and `cloud-identity.ts`, with
  `mirror-sync.ts` for state sync.
- **Pairing:** QR code claim via `claim-qr.ts`, device PIN flow, desktop
  discovery.
- **Memory:** `memory-graph-sim.ts` for in-memory graph simulation,
  `memory-cache.ts` for caching.
- **State:** Zustand 5.0.8, `@tanstack/react-query` 5.90.2 for data fetching.
- **Storage:** `react-native-mmkv` for key-value, `expo-sqlite` for SQLite.
- **Styling:** `tailwind-merge` + `clsx` for class merging.

## Entry and build

```bash
cd apps/iOS-client
bun install
cp .env.example .env
npx expo prebuild --platform ios
cd ios && LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 pod install && cd ..
open ios/Alfred.xcworkspace
```

In Xcode: select target **Alfred**, set Signing Team, Run. Metro (JS reload)
runs separately: `npx expo start --dev-client`.

## Publishing

EAS Build for App Store:

```bash
EAS_NO_VCS=1 EAS_PROJECT_ROOT="$(pwd)" eas build --profile preview --platform ios
```

App Store review notes in `APP_STORE.md` must state the companion Mac
requirement.

## Voice integration

The app supports two voice stacks:

- **Cascade** (Deepgram STT → LLM → ElevenLabs TTS) — via the Mac backend
- **GPT-Live** — direct GPT-Live WebRTC connection, with captions on
  `alfred.caption` / `alfred.user` LiveKit data topics

`verify-livekit-natives.sh` confirms WebRTC framework integration in the
Simulator build. JS falls back to text-only Talk when `isLiveKitAvailable()`
returns `false` (`no-sdk` state).

## Robot integration

`robot-api.ts` communicates with AlfredBot on the local network for expression
control. `bot-qr.ts` generates QR codes for robot pairing.

## Architecture notes

The app is a full-featured mobile client, not a thin wrapper. It runs:
- Voice/audio through LiveKit + CallKit
- Photo capture through VisionCamera + Expo Camera
- Calendar access through Expo Calendar module
- Notifications through Expo Notifications
- Background fetch for periodic sync
- File system access for document handling

The desktop API surface (`desktop-api.ts`) communicates with the Mac's HTTP
API for pairing, token minting, and session management — the same API documented
in [desktop-text.md](desktop-text.md).

## Related

- [desktop-text.md](desktop-text.md) — the HTTP API it connects to
- [voice-agent.md](voice-agent.md) — the voice stack
- [alfredbot.md](alfredbot.md) — the parallel robot surface, same pairing
  shape