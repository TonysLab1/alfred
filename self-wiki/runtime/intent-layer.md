# The Intent Layer — How Tool Calls, Delegation, and Generation Are Dispatched

Between the assembled prompt and the specific action Alfred takes, a layer of
intent recognition determines the control flow. This document covers the two
mechanisms: conversational tool dispatch and the early delegation exits.

## Conversational tool dispatch

In the cascade voice path, the LLM calls tools by name. `applyConversationalTool`
in `packages/core/src/conversational-tools.ts` is the dispatch hub:

```ts
export async function applyConversationalTool(
  toolName: string | undefined,
  args: Record<string, unknown>,
  ports: ConversationalPorts | undefined,
  dueReminders: DueReminderSummary[],
): Promise<{ mode: "replace" | "ack"; speech: string } | null>
```

It returns one of two modes:

- **`replace`** — the tool's result replaces whatever the model said. Used for
  weather, news, markets, time, situational, and lights tools, where the model's
  lead-in ("let me check") is discarded and the authoritative data is spoken
  instead.
- **`ack`** — the tool acknowledges the action. Used for memory writes
  (`remember_memory`, `write_node`) and reminders (`update_reminder`), where the
  model may have already spoken a sentence and the tool just confirms.

If no ports are configured for the requested tool, `null` is returned — the call
is silently ignored.

## The ConversationalPorts interface

`packages/core/src/ports.ts` defines the contract. `ConversationalPorts` bundles
optional port implementations:

```ts
interface ConversationalPorts {
  reminders?: ReminderPort;
  structuredMemory?: StructuredMemoryPort;
  weather?: WeatherForecastPort;
  news?: NewsPort;
  markets?: MarketsPort;
  currentTime?: CurrentTimePort;
  situational?: SituationalPort;
  lights?: StudioLightsPort;
}
```

Each port wraps an actual implementation — API calls to weather services, the
OIP memory provider, the briefing controller, the Elgato lights controller.
Whether a tool is available depends entirely on which ports the surface has
registered.

## Capability discovery

`conversationalCapabilities(ports)` returns the list of tool names that are
actually available in the current session, based on which ports are registered.
This list drives the capability guidance block in `PromptAssembler` — the LLM
learns which tools exist by reading the guidance, not by inspecting the tool
schema.

`conversationalToolSchemas(ports)` returns the Zod-validated tool parameter
schemas for the available tools. These are registered with the LLM provider as
the formal tool definitions.

The `situational` port enables six tools at once: earthquakes, weather alerts,
space weather, natural events, exchange rate, and hacker news. These are in a
single port because they are all read-only lookups with similar failure
modes.

## The early delegation exits

Before any prompt is assembled, `SessionOrchestrator` checks three parsers at
`session.ts:370-382`:

### `parseDelegateIntent(text)`

Matches the literal syntax `delegate:category:description`. This is the
simulator's shortcut — a text format that short-circuits directly to
`runDelegation`. Not a natural language parser.

### `looksLikeDocsIngestTask(text)`

`packages/core/src/docs-ingest-intent.ts`: regex heuristics for documentation
ingestion requests:

- `(list|show|what) … docs?/documentation … folders?`
- `ingest my docs`
- `ingest|import|index|update|scan|process|watch … docs?|documentation|markdown`
- `docs?/documentation folder … ingest|watch|index|scan|update`

If any match, the turn routes to `runDelegation(turn, "research", text)` without
building a normal prompt.

### `looksLikeXIngestTask(text)`

`packages/core/src/x-ingest-intent.ts`: regex heuristics for X/Twitter or
YouTube ingestion:

- Absolute URLs to `x.com` or `twitter.com`
- Absolute URLs to `youtube.com` or `youtu.be`
- `ingest … x|twitter|youtube|yt|notes?`
- `pull … x|twitter|youtube|yt|link|post|article|thread|video`

If matched, routes to `runDelegation(turn, "research", text)`.

## The additional intent matchers (not wired as early exits)

`packages/core/src/` contains a family of `*intent.ts` files with unit tests:

| File | Matches | Used by |
|---|---|---|
| `news-intent.ts` | Headline and article summarization requests | The cascade voice stack (not the core orchestrator) |
| `markets-intent.ts` | Crypto and metals price queries | Cascade voice |
| `situational-intent.ts` | Earthquake, alert, space weather, natural event, FX, HN queries | Cascade voice |
| `time-intent.ts` | Current time or date requests | Cascade voice |
| `weather-intent.ts` | Weather forecast requests | Cascade voice |
| `studio-lights-intent.ts` | Light control commands | Cascade voice |

These are **not called inside `SessionOrchestrator`.** They live in the cascade
voice path and are used for intent detection when the voice model dispatches
through the LiveKit stack rather than the standard text orchestrator. For text
turns through the orchestrator, the LLM calls tools directly by name — no
regex-based intent matching is needed.

This distinction matters: the text path uses tool-calling; the voice path uses
regex-based intent on top of tool-calling. The wiki's tool-surface doc
([../capabilities/tool-surface.md](../capabilities/tool-surface.md)) documents
the tools themselves; this doc documents the dispatch layer that decides which
path invokes them.

## Why there's no policy gate here

`applyConversationalTool` executes immediately when called. There is no safety
check, no grade evaluation, no prominence or urgency assessment between
receiving a `control_studio_lights` call and switching the lights. The dispatch
layer is purely mechanical — match tool name, call the port, return the speech.

This is documented as a stance observation rather than a defect, because the
system architecture — one user, one trust level, a surface that requires
physical proximity — makes a gate unnecessary for the current deployment. See
[../capabilities/README.md](../capabilities/README.md) for the grade
documentation.

## Related

- [turn-lifecycle.md](turn-lifecycle.md) — where the early exits fire
- [prompt-assembly.md](prompt-assembly.md) — where capability guidance is built
- [../capabilities/tool-surface.md](../capabilities/tool-surface.md) — the
  full tool inventory
- [../capabilities/delegation.md](../capabilities/delegation.md) — where
  delegation routes