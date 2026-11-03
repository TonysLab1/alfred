# Voice Agent — Cascade and GPT-Live Paths

The voice agent at `apps/voice-agent/src/` is the primary surface. It supports
two operating modes: a cascade STT→LLM→TTS pipeline and an experimental GPT-Live
realtime model.

Entry point: `apps/voice-agent/src/brain.ts`.

## Cascade voice

```
Deepgram STT → OpenAI/Grok LLM → ElevenLabs TTS
```

The cascade path uses the standard `SessionOrchestrator` from `packages/core`
and a `MemoryController` with both JSONL and OIP providers.

Persona: loaded from disk via `ensureAndLoadPersona(profileId)`, with the
HYPATIA override when `ALFRED_ARCHIVE_URL` is set.

## GPT-Live voice

```
OpenAI GPT-Live realtime model (experimental alpha)
```

The live path uses a separate instruction builder at
`apps/voice-agent/src/live/instructions.ts` rather than `PromptAssembler`. It
splits instructions into two blocks:

- **Voice instructions** — the top-level spoken interaction model. Includes
  barge-in rules, filler phrases, expression cues, and delegation guidance.
- **Backend instructions** — the work-handling model that receives delegated
  calls and uses tools.

Persona: the same `LoadedPersonaContext` from `brain.persona` is pushed into
both instruction blocks, replicating the SOUL/IDENTITY/USER pattern.

### Barge-in

The GPT-Live model owns barge-in. When the user cuts in, the model acknowledges
briefly ("got it") and answers the new point. The instruction in
`buildLiveVoiceInstructions` is explicit:

> "You listen while speaking; the model owns barge-in. Acknowledge briefly when
> the user cuts in, then answer the new point."

### Robot expressions

`control_studio_lights` handles lighting; `show_expression` drives the
AlfredBot face. The voice instructions guide expression use:

> "When speaking from the robot, use show_expression for a smile, frown, wink,
> nod, tilt, or wave that matches the turn. Do not emote on every sentence.
> Never perform angry as aggressive motion."

## Tool surface

The full live tool set is listed in
[../capabilities/tool-surface.md](../capabilities/tool-surface.md). Key tools
unique to or preferred in the voice path:

- `play_daily_briefing` — the proactive spoken greeting
- `decline_briefing_offer` — the soft-offer refusal
- `show_expression` — robot face control

## Delegation

The routing rules in `brain.ts` differ from the desktop path (see
[../capabilities/delegation.md](../capabilities/delegation.md)). The voice
agent uses `harness.codex` for coding rather than `harness.openclaw`, and does
not register `household` or `general` categories.

## Reminders and briefing

The briefing controller lives on `AlfredBrain.briefing` and is constructed from
OIP reminders and an agenda port. Reminders flow from the OIP reminders table
into `list_due_reminders` and the briefing greeting.

## Session construction per call

In the cascade path: each voice turn creates or reuses a `SessionOrchestrator`.
In the GPT-Live path: the session is a persistent WebSocket connection with
transcript deltas, not discrete turns.

Both paths share: `brain.persona`, `brain.memory`, `brain.agents`,
`brain.briefing`, `brain.lights`, `brain.archive` (when connected).

## Related

- [../identity/persona-override.md](../identity/persona-override.md) — the
  HYPATIA branch that lives here
- [../runtime/prompt-assembly.md](../runtime/prompt-assembly.md) — the parallel
  assembly path for GPT-Live
- [../capabilities/delegation.md](../capabilities/delegation.md) — routing rules
- [../capabilities/tool-surface.md](../capabilities/tool-surface.md) — the tool
  set