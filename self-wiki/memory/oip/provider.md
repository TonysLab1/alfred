# OIP Provider — The API Surface and Locking Model

`OipLocalMemoryProvider` in `packages/memory/src/oip-local/provider.ts` (588
lines) implements the `MemoryProvider` contract. It wraps `PackageStore`,
`ArtifactStore`, `SqliteMemoryIndex`, and `VectorIndex` into a single API.

## Construction and initialization

```ts
constructor(
  readonly rootDir: string,
  id = OIP_LOCAL_MEMORY_PROVIDER_ID,
  vectors?: VectorIndex,
)
```

Default `rootDir` = `data/memory-oip/<profileId>/` from `defaultOipMemoryRoot()`.

The `vectors` parameter is injected; omitting it defaults to `NoopVectorIndex`.
This is the sole entry point for swapping in a real vector index:

```ts
this.vectors = vectors ?? new NoopVectorIndex();
```

`ensureReadyUnlocked()` runs package-store root creation (mkdir + storage-format
bootstrapping) and a conditional index rebuild.

## The exclusive lock

The provider serializes operations through a promise chain:

```ts
private opChain: Promise<void> = Promise.resolve();
```

`runExclusive(fn)` queues work onto this chain. Nested calls from the same
exclusive turn are allowed (tracked by `opDepth`) — this is specifically
because `commitTurn` calls `writeConversationalMemory` which calls
`rebuildIndexes`, and all three must share the same SQLite handle.

Without this, a concurrent `rebuildIndexes` could close the database while
another operation held a prepared statement. The comments are explicit: this
pattern exists because "rebuild cannot close the DB under another op."

## Core API: commitTurn

Covered in [../conversation-capture.md](../conversation-capture.md). In brief:

1. runs inside exclusive lock
2. creates an Observation package
3. indexes it incrementally
4. runs conversational memory extraction
5. if structured output is found, writes it and rebuilds

## Core API: retrieve

Delegates to `retrieveMemories()` — covered in full in
[../retrieval.md](../retrieval.md).

## Core API: record CRUD

The provider exposes:

```ts
createRecord(type, body, logicalId?, opts?) → MemoryRevision
updateRecord(didOrLogicalId, patch, opts?) → MemoryRevision
getRecord(id) → MemoryRevision | null
deleteRecord(didOrLogicalId) → void
```

All go through the package store. `createRecord` and `updateRecord` both accept
a `reindex` option (default `true`) that triggers `indexRevisionUnlocked()`.

## Index management

`indexRevisionUnlocked(revision)` indexes a single record by:

- upserting into records table
- rebuilding FTS5 for that record
- upserting edges from drefs (subject/object/supersedes/contradicts/reinforces)
- upserting temporal and reminder rows when present
- calling `vectors.upsert()` (no-op by default)

`rebuildIndexesUnlocked()` does a full rebuild: regenerates the SQLite schema,
re-reads all current revisions from the package store, and indexes every one.
This is called on initialisation if the index DB is absent.

## Artifact management

The `ArtifactStore` handles binary objects:

```ts
putArtifactBytes(bytes, opts) → ArtifactRecord
getArtifactBytes(id) → Buffer | null
getArtifactPath(id) → string
```

Artifacts are content-addressed: the hash determines the storage path at
`artifacts/sha256/<ab>/<cd>/sha256-<hex>`. A row is recorded in the SQLite
artifacts table. Updates reuse the existing artifact when the hash matches.

## Integrity

`verifyStore()` in `integrity.ts` checks manifest-revision consistency,
detects orphaned revisions, and verifies content hashes. `rebuildIndexes()`
is the recovery path when the index DB is out of sync with the package store.

## Reminder interface

The provider exposes a `DueReminder` type and queries against the reminders
table. The briefing controller and `list_due_reminders` tool use this path.

## The MemoryProvider contract

`OipLocalMemoryProvider` implements `MemoryProvider` from `@alfred/contracts`,
which means it satisfies:

```ts
interface MemoryProvider {
  manifest: MemoryProviderManifest;
  initialize(): Promise<void>;
  retrieve(query: MemoryQuery): Promise<MemoryRetrievalResult>;
  commitTurn(commit: MemoryTurnCommit): Promise<void>;
  inspect?(...): Promise<...>;    // optional
  edit?(...): Promise<...>;       // optional
  delete?(...): Promise<void>;    // optional
  importCanonical?(...): Promise<...>; // optional
  exportCanonical?(...): Promise<...>; // optional
}
```

The manifest advertises: retrieve, commit_turn, inspect, edit, delete,
import_canonical, export_canonical, file_backed, episodic, temporal_graph.

## Related

- [record-model.md](record-model.md) — what records look like
- [indexes.md](indexes.md) — the SQLite tables
- [../conversation-capture.md](../conversation-capture.md) — the commitTurn path