# OIP SQLite Indexes — The Tables Behind Search and Graph Walk

The index database is a SQLite file at
`data/memory-oip/<profileId>/indexes/records.db`. Its schema is defined in
`packages/memory/src/oip-local/indexes/sqlite-index.ts` (736 lines).

## Tables

### records

```sql
CREATE TABLE IF NOT EXISTS records (
  id           TEXT PRIMARY KEY,
  logical_id   TEXT NOT NULL,
  current_revision TEXT NOT NULL,
  record_type  TEXT NOT NULL,
  schema_type  TEXT,
  name         TEXT,
  owner_id     TEXT,
  visibility   TEXT,
  created_at   TEXT,
  updated_at   TEXT,
  learned_at   TEXT,
  search_text  TEXT
);
```

`id` is the `did:memory:<logicalId>` string. `search_text` is populated during
indexing from the record's name, text, schema.name, and schema.description
fields — this is the column FTS5 indexes.

Indexes on records:
- `idx_records_type ON records(record_type)` — for type-filtered scans

### revisions

```sql
CREATE TABLE IF NOT EXISTS revisions (
  record_id        TEXT NOT NULL,
  revision_hash    TEXT NOT NULL,
  previous_revision TEXT,
  canonical_path   TEXT NOT NULL,
  created_at       TEXT,
  PRIMARY KEY (record_id, revision_hash)
);
```

Tracks which revision files exist and what they supersede. The `canonical_path`
points to the on-disk JSON file.

### edges

```sql
CREATE TABLE IF NOT EXISTS edges (
  source_id        TEXT NOT NULL,
  predicate        TEXT NOT NULL,
  target_id        TEXT NOT NULL,
  source_revision  TEXT,
  PRIMARY KEY (source_id, predicate, target_id, source_revision)
);
```

A flat triple store. The graph walk in `GraphIndex.expand()` does BFS by
querying this table with `edgesFrom(source_id)` and `edgesTo(target_id)`.

No foreign key constraints — edges can reference records that don't exist in the
records table, so the index can be rebuilt independently.

Indexes on edges:
- `idx_edges_source ON edges(source_id)`
- `idx_edges_target ON edges(target_id)`
- `idx_edges_pred ON edges(predicate)`

### records_fts (FTS5 virtual table)

```sql
CREATE VIRTUAL TABLE records_fts USING fts5(
  record_id UNINDEXED,
  name,
  search_text,
  record_type UNINDEXED
);
```

`record_id` and `record_type` are `UNINDEXED` so they store but don't tokenise.
The content columns `name` and `search_text` are what FTS5 matches against.

Retrieval queries this with the bm25 ranking function:

```sql
SELECT record_id, bm25(records_fts) AS rank
FROM records_fts
WHERE records_fts MATCH ?
ORDER BY rank
LIMIT ?
```

The match string is built by `toFtsQuery()` which lowercases, splits on
non-alphanumeric characters, drops stopwords, and wraps each token in
double-quotes joined with `OR`. So a query "turn lifecycle" becomes:

```sql
MATCH '"turn" OR "lifecycle"'
```

No stemming, no prefix matching. Exact token match only.

### artifacts

```sql
CREATE TABLE IF NOT EXISTS artifacts (
  hash          TEXT PRIMARY KEY,
  mime_type     TEXT,
  byte_size     INTEGER,
  stored_path   TEXT,
  created_at    TEXT
);
```

Content-addressed artifact registry. The `stored_path` is relative to the
artifacts root; the hash is the sha256 of the blob.

### temporal

```sql
CREATE TABLE IF NOT EXISTS temporal (
  record_id     TEXT PRIMARY KEY,
  valid_from    TEXT,
  valid_until   TEXT,
  learned_at    TEXT,
  event_start   TEXT,
  event_end     TEXT
);
```

Used for time-window queries in retrieval — the `isWithin(...)` methods in the
index scan this table.

### reminders

```sql
CREATE TABLE IF NOT EXISTS reminders (
  record_id             TEXT PRIMARY KEY,
  remind_at             TEXT,
  remind_at_sort_key    TEXT,
  reminder_status       TEXT,
  reminder_reason       TEXT,
  reminder_timezone     TEXT,
  reminder_snoozed_until TEXT
);
```

The reminder subsystem uses this table, plus the `reminder*` fields on
revisions.

## How the index is populated

When `indexRevisionUnlocked(revision)` is called:

1. the `records` table gets an `INSERT OR REPLACE` for the record row
2. the old FTS5 row for that record_id is deleted and a new one inserted —
   FTS5 has no `REPLACE` support
3. the record's drefs are scanned for subject/object/supersedes/contradicts/
   reinforces DIDs, and each pair generates an edge row
4. `temporal` and `reminders` tables are upserted when the record carries
   the relevant fields
5. `vectors.upsert(recordId, revision, text, embedding)` is called — a no-op
   by default

## Why the rebuild path exists

`rebuildIndexesUnlocked()` in the provider method completely resets the index
by deleting and recreating the SQLite file, then running
`indexRevisionUnlocked()` for every current revision from the package store.

This is called:

- on initialisation when the index DB doesn't exist
- when the user runs `pnpm memory -- oip-rebuild`
- inside `writeConversationalMemory()` after structured writes that produce new
  edges

The rebuild-after-write pattern in conversation memory explains the exclusive
lock: committing a conversational turn can trigger a full index rebuild,
and that rebuild must not race with retrieval running on the same turn.

## What the indexes don't store

- **No full-text indexing of artifact content** — only metadata
- **No indexes on drefs or alfred fields** — those are queried by record-type
  scans with in-memory filtering
- **No foreign keys** — edges don't validate targets
- **No vector index populated by default** — the `NoopVectorIndex` does nothing

## Related

- [../retrieval.md](../retrieval.md) — how these indexes are queried
- [record-model.md](record-model.md) — the records the indexes point at
- [provider.md](provider.md) — the API that populates the indexes