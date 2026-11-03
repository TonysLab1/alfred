# Desktop Text — HTTP API and Sidecar Mode

The desktop text surface at `apps/desktop-client/src/` provides an HTTP API
for text conversations and a sidecar mode for integration with other local
processes.

Entry points: `apps/desktop-client/src/lib/text-session.ts` for session
construction, `apps/desktop-client/src/routes/conversation.ts` for the HTTP
routes.

## Session model

Sessions are keyed by conversation key parts (household id, device token, etc.)
and memoised in a module-level `Map`:

```ts
const runtimes = new Map<string, { session: SessionOrchestrator; … }>();
```

`getTextSession(parts)` returns an existing runtime or builds a new one. This
is the caching that freezes persona after construction — see
[../identity/persona-loading.md](../identity/persona-loading.md).

`resetTextSession(parts)` cancels the session and deletes the runtime, forcing
a rebuild on the next `getTextSession`.

## Routes

Defined in `apps/desktop-client/src/routes/conversation.ts`:

| Route | Purpose |
|---|---|
| `POST /api/conversation/turn` | Process a text turn: JSON or NDJSON streaming |
| `POST /api/conversation/cancel` | Cancel the active turn |
| `POST /api/conversation/reset` | Reset the session (delete and rebuild runtime) |
| `GET /health` | Return session count and OpenAI key status |

The text-turn route streams the response as NDJSON, supporting both text-only
and tool-call interleaving.

## Provider setup

Only OIP is registered as a memory provider:

```ts
const oip = oipForProfile(profileId);
memory.register(oip);
await memory.initialize(OIP_LOCAL_MEMORY_PROVIDER_ID);
```

No JSONL provider — the desktop path has a simpler memory surface.

## LLM registration

The desktop path registers available LLM providers based on environment:

```ts
if (openaiKey) register(OpenAiResponsesLLMProvider);
if (grokKey) register(XaiGrokLLMProvider);
```

If neither is available, a `FakeLLMProvider` is registered that echoes back the
user's text. This allows the surface to start and serve health checks even
without LLM credentials.

## Tool surface

Ports registered on the `SessionOrchestrator`'s conversational ports:

- reminders: Apple Reminder port backed by the briefing controller
- structured memory: writes to OIP via `writeConversationalMemory`
- weather: live forecast lookup
- news: headline retrieval
- markets: crypto and metals prices
- lights: Elgato lights controller

These are wired to `conversational-tools.ts` which provides the intent-match
layer. The full live voice tool set (weather alerts, earthquakes, space weather,
natural events, Hacker News, expressions) may not all be present.

## Delegation rules

See [../capabilities/delegation.md](../capabilities/delegation.md). The desktop
path uses different chains from the voice agent — notably `harness.openclaw` for
coding instead of `harness.codex`, and additional `household`/`general`
categories.

## STT/TTS

The desktop path registers fake STT and TTS providers:

```ts
registry.registerStt(new FakeSTTProvider("stt.desktop.fake"));
registry.registerTts(new FakeTTSProvider("tts.desktop.fake"));
```

So the desktop surface is text-only — it reuses the orchestrator's speech
path internals but the STT and TTS layers are stubs.

## Sidecar mode

Started with `ALFRED_SIDECAR_MODE=true`, the desktop client runs as an HTTP
server on the configured `PORT` (default 3000) without a web UI. This is the
mode for HomeBuddy integration and other local process communication.

Enable with:
```bash
ALFRED_SIDECAR_MODE=true pnpm desktop
```

## Related

- [../identity/persona-override.md](../identity/persona-override.md) — HYPATIA
  on the desktop path does not replace persona files, creating a surface
  discrepancy
- [voice-agent.md](voice-agent.md) — the voice path for comparison
- [../memory/providers.md](../memory/providers.md) — provider setup