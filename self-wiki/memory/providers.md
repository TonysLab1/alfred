# Memory Providers — The Two Backends

Alfred registers two memory providers in every session. `MemoryController`
routes retrieval and commit calls to one active provider, while structured
writes always go to OIP regardless.

## The controller

`packages/memory/src/controller.ts` implements `MemoryControllerPort` and is a
thin routing layer.

- `register(provider)` adds a provider keyed by `manifest.id`
- `initialize(preferredId?)` sets the active provider from a persisted setting,
  falling back to the first registered
- `retrieve(query)` delegates to the active provider
- `commitTurn(commit)` delegates to the active provider

Only one provider is active at a time. The setting is persisted through
`MemorySettingsRepository`.

## LocalFileMemoryProvider (JSONL)

A flat JSONL file per profile, at `data/memory/<profileId>.jsonl` (configurable
with `ALFRED_MEMORY_LOCAL_PATH`).

Strengths:
- no index build time; startup is instant
- fast O(n) keyword scan for small corpora
- human-readable on disk: one JSON object per line

Weaknesses:
- no structured types — everything is a flat memory entry
- no revision chain; updates overwrite
- no relationship model; no graph walk
- linear scan on retrieval; falls over at scale

The provider id is `memory.local`.

## OipLocalMemoryProvider (OIP)

The structured provider, at `data/memory-oip/<profileId>/`. This is the
provider that receives every structured write, every conversation turn commit,
and every document ingestion output.

Provider id: `memory.oip-local`. Exported constant:
`OIP_LOCAL_MEMORY_PROVIDER_ID`.

Manifest capabilities: retrieve, commit_turn, inspect, edit, delete,
import_canonical, export_canonical, file_backed, episodic, temporal_graph.

The full interna are documented under [oip/](oip/README.md). What matters at the
provider-registration level is that this provider carries:

- a `PackageStore` for file-system package storage with revision chaining
- an `ArtifactStore` for binary blobs
- a `SqliteMemoryIndex` for records, edges, FTS5, reminders, and temporal data
- an optional `VectorIndex`, defaulting to `NoopVectorIndex`

## Active provider assignment

In the voice agent at `apps/voice-agent/src/brain.ts`:

```ts
const memory = new MemoryController(config.profile.id, persistence.memorySettings);
memory.register(localMemory);
memory.register(oipMemory);
await memory.initialize(memoryProviderId);
```

`memoryProviderId` comes from `ALFRED_MEMORY_PROVIDER_ID` in `.env`
(`memory.oip-local`), so OIP is the active provider.

In the desktop text session at `apps/desktop-client/src/lib/text-session.ts`,
only OIP is registered:

```ts
const memory = new MemoryController(profileId, persistence.memorySettings);
const oip = oipForProfile(profileId);
memory.register(oip);
await memory.initialize(OIP_LOCAL_MEMORY_PROVIDER_ID);
```

Both surfaces converge on OIP as the active provider for the deployed
configuration. The JSONL provider is registered but not set as active — it is a
fallback option, not the default.

## The implicit "always OIP" rule

While the controller routes retrieval and commitTurn to the active provider,
structured memory writes (`remember_memory`, ingestion pipelines) bypass the
controller entirely and construct their own OIP provider instance. This is
because `remember_memory` is defined in voice-agent tool code that has direct
access to the OIP provider, and document ingestion similarly constructs an
`OipLocalMemoryProvider` from the profile root directory.

So the rule is not "the active provider handles everything." It is:

- **Conversational commitTurn + retrieval:** active provider (OIP)
- **Structured writes (entities, assertions, ingest):** always OIP, never the
  JSONL provider

## Related

- [conversation-capture.md](conversation-capture.md) — the commitTurn
  implementation
- [retrieval.md](retrieval.md) — the retrieve implementation
- [oip/README.md](oip/README.md) — the OIP provider internals