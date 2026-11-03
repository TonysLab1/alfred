# OIP Internals — The Object Information Protocol Data Layer

This area covers the storage substrate behind `OipLocalMemoryProvider`. It
describes the data model, the package-revision system, the on-disk layout, the
SQLite index schema, and the record lifecycle.

## Documents in this area

| Document | What it covers |
|---|---|
| [record-model.md](record-model.md) | The five OIP types, the revision schema, drefs, integrity, and the physical layout |
| [provider.md](provider.md) | `OipLocalMemoryProvider`: the API surface, exclusive locking, and the `MemoryProvider` contract |
| [indexes.md](indexes.md) | The SQLite schema: records, revisions, edges, FTS5, artifacts, temporal, reminders |

## Key things to know before reading deeper

**OIP uses content-addressed revision files.** A record has a stable logical
id (ULID) and a `did:memory:<id>`. Revisions are files keyed by
`sha256:<hex>` of the canonical JSON. The word "revision" here means
"revision of the memory object," not "revision of a document." See
[record-model.md](record-model.md).

**FTS not vectors.** The retrieval index is SQLite FTS5 with bm25 scoring.
The `VectorIndex` exists as an interface with a `NoopVectorIndex` default.
See [indexes.md](indexes.md) and [../retrieval.md](../retrieval.md).

**Edges are a flat triple table.** `source_id, predicate, target_id` plus a
source revision. No graph database, just BFS over a SQLite table. See
[indexes.md](indexes.md).

**Integrity is verifiable and rebuildable.** `integrity.ts` walks the on-disk
package store, checks manifest-revision consistency, and detects orphans. The
provider exposes `verifyStore()` and `rebuildIndexes()`.

**Conversation capture is part of OIP, not a separate store.** Every turn
becomes an Observation, indexed alongside document chunks and extracted
entities. The same FTS table covers everything. See
[../conversation-capture.md](../conversation-capture.md).

## Related

- [../README.md](../README.md) — the broader memory area
- [../retrieval.md](../retrieval.md) — the retrieval chain that queries these
  indexes
- [../docs-ingestion.md](../docs-ingestion.md) — the ingestion pipeline that
  produces records
- [../providers.md](../providers.md) — the provider routing context