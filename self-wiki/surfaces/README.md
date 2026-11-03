# Surfaces — How Alfred Is Reached

Alfred runs across six different front surfaces, each with its own transport,
persona injection path, and tool surface. This area catalogs them.

All six share the same core (packages/core, packages/memory, packages/agents)
but differ in their session construction and delivery mechanism.

## Documents in this area

| Document | What it covers |
|---|---|
| [voice-agent.md](voice-agent.md) | Cascade STT→LLM→TTS and GPT-Live realtime paths |
| [desktop-text.md](desktop-text.md) | HTTP API and sidecar mode for text conversations |
| [simulator.md](simulator.md) | CLI test harness for turn simulation |
| [voice-client.md](voice-client.md) | Browser-based voice frontend (LiveKit relay) |
| [alfredbot.md](alfredbot.md) | Discord/social bot connector |
| [ios-client.md](ios-client.md) | Mobile app with pairing, photo capture, cloud sync |

## The key differences between surfaces

### Persona loading

| Surface | Loads from disk? | Can be overridden? | Override trigger |
|---|---|---|---|
| Voice agent | Yes — `ensureAndLoadPersona(profileId)` | **Yes — entire persona replaced** | `ALFRED_ARCHIVE_URL` |
| Desktop text | Yes — `loadPersonaContext(dir)` | **Partial — only `systemInstructions` replaced** | `ALFRED_ARCHIVE_URL` |
| Simulator | Unknown | Unknown | Unknown |

See [../identity/persona-override.md](../identity/persona-override.md) for the
full override mechanics and the surface disagreement it creates.

### Tool surface

The live voice path has the richest tool set, listed in
[../capabilities/tool-surface.md](../capabilities/tool-surface.md). The desktop
text path provides a subset through port wiring in `text-session.ts`. The
simulator and other surfaces may not register the same tools.

### Memory provider

| Surface | Active provider | Structured writes |
|---|---|---|
| Voice agent | OIP (configurable) | Always OIP |
| Desktop text | OIP (only provider registered) | Always OIP |
| Simulator | Configurable | Unknown |

See [../memory/providers.md](../memory/providers.md) for routing.

## What the surfaces have in common

The `AlfredBrain` type in `apps/voice-agent/src/brain.ts:104` is the shared
interface:

```ts
interface AlfredBrain {
  config: {...};
  persona: LoadedPersonaContext;
  memory: MemoryController;
  agents: AgentRouter;
  briefing: BriefingController;
  lights: ElgatoLightsController;
  archive?: HttpArchivePort;
  ...
}
```

Every surface constructs or receives an equivalent object, though some omit
components (simulator may lack lights; desktop text has its own briefing
construction).

## Related

- [../runtime/turn-lifecycle.md](../runtime/turn-lifecycle.md) — the shared
  orchestrator
- [../identity/persona-loading.md](../identity/persona-loading.md) — where
  persona enters each surface
- [../capabilities/README.md](../capabilities/README.md) — the tools available