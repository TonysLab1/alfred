# OIP Record Model — Types, Revisions, and On-Disk Layout

How a memory record is structured, versioned, and stored. Every fact Alfred
remembers, from a document chunk to a conversation turn to a person entity, is
an instance of this model.

## The five record types

`packages/memory/src/oip-local/schemas.ts:4`:

```ts
export const MemoryRecordTypeSchema = z.enum([
  "Entity",
  "Episode",
  "Assertion",
  "Observation",
  "Artifact",
]);
```

| Type | What it is | Examples in Alfred |
|---|---|---|
| Entity | A person, place, organization, thing, or concept | Person records, org records, the docs-folder Collection |
| Episode | A dated event or interaction | Not currently used in the commitTurn path |
| Assertion | A subject-predicate-object statement | "Morgan shares an apartment in Portland" |
| Observation | A captured piece of text tied to a time | Conversation turns, document sections, ingested note excerpts |
| Artifact | A binary blob with content hash and metadata | The raw bytes of an ingested markdown file, a PDF |

The four non-Artifact types (memory-bearing types) share the same revision
schema; only convenience fields differ (Assertion has `subject`/`predicate`/
`object`, Observation has `observedAt`, etc.).

## The revision schema

`MemoryRevisionSchema` in `schemas.ts:32` is a Zod-validated, passthrough
schema. The core fields:

- `id: string` — always starts with `did:memory:`
- `type: MemoryRecordType`
- `revision: string` — the content hash of this revision
- `previousRevision: string | null`
- `schemaType: string?` — e.g. `"https://schema.org/CreativeWork"`
- `schema: Record<string, unknown>` — the typed schema.org block
- `alfred: AlfredMetaSchema` — Alfred-specific metadata namespace
- `drefs: Record<string, unknown>` — relationship pointers
- `provenance: ProvenanceSchema` — source tracking
- `createdAt, updatedAt, learnedAt: string` — timestamps
- `supersedes, contradicts, reinforces: string[]?` — graph relationship DIDs
- `name, text: string?` — convenience fields

Plus type-specific convenience fields: `subject`, `predicate`, `object`,
`participants`, `location`, `validTimeStart/End`, `speaker`, `observedAt`,
`sourceArtifact`, `contentHash`, `mimeType`, `byteSize`, `originalFilename`,
`storedAt`, `ingestedAt`, and reminder fields.

The schema is `passthrough`, so additional keys on a revision survive Zod
validation — they pass through without being dropped.

## The alfred namespace

`AlfredMetaSchema` in `schemas.ts:23`:

```ts
{
  entityClass?: string;          // e.g. "Person", "docs_folder", "docs_file"
  confidence?: number;            // 0–1
  confidenceLabel?: "confirmed" | "high" | "medium" | "low" | "inferred";
  visibility?: "private" | "shared" | "public";
  owner?: string;
  assertionType?: "explicit" | "inferred" | "extracted";
}
```

Also passthrough — the `toolcallCount` and `reasoningLineCount` set by
`commitTurn` live in this namespace.

The `self-identity.ts` resolver uses `alfred.isSelf === true` to determine which
Entity is Alfred's own user.

## drefs — the relationship vocabulary

`drefs` is a free-form `Record<string, unknown>`. No schema constrains it beyond
that, but a vocabulary has emerged in practice:

| dref key | Convention | Example |
|---|---|---|
| `session` | The session id this record belongs to | `"sess_abc123"` |
| `turnData` | Compound pointer to the turn | `"sess_abc123/turn_xyz"` |
| `isPartOf` | The containing record(s) | `[fileDid, folderDid]` |
| `sourceArtifact` | The artifact this observation was extracted from | `"did:memory:..."` |
| `subject` / `object` | Assertion subject and object DIDs | `"did:memory:..."` |

The graph expansion in retrieval uses edges from the edge table, not drefs
directly. So drefs are metadata on the record, not the walk graph.

## The supersedes / contradicts / reinforces triad

Three array fields on every revision. These are OIP-level relationship links:

- `supersedes: string[]` — previous records this one replaces
- `contradicts: string[]` — records this one disagrees with
- `reinforces: string[]` — records this one supports

These are first-class relationships, not just informal text. When a document
section is removed on re-ingest, the old Observation is superseded by the new
empty one — the old record does not float in the index.

## Content-addressed revisions

The full record lifecycle is in the `PackageStore` (`package-store.ts`):

### Creating a record

1. a logical id is generated (ULID) or provided
2. the body is assembled with all fields defaulted (empty objects for schema,
   alfred, drefs, provenance)
3. the complete record (minus `revision`) is canonicalised to JSON and hashed
4. the hash becomes the `revision` field and the filename:
   `packages/<logicalId>/revisions/sha256-<hex>.json`
5. a manifest is written at `packages/<logicalId>/manifest.json` containing
   `{id, type, currentRevision, createdAt, updatedAt}`
6. a README and an empty `artifact-refs.json` are created per package

### Appending a revision

1. the current revision is read
2. the patch is shallow-merged (schema, alfred, drefs, provenance are merged
   keywise; the rest overwritten)
3. `previousRevision` is set to the old revision hash
4. the merged result is hashed
5. if the hash matches an existing revision file, the call is a no-op — same
   content, same hash, existing record returned
6. otherwise the new revision is written and the manifest updated

So "update" does not mean "edit." It means "append a new revision that points
back to the previous one." Every revision ever written remains on disk.

### Deleting a record

`deletePackage(logicalId)` removes the package directory entirely. So deletion
is a real destructive operation, not a soft deprecation.

## Physical layout

Under `data/memory-oip/<profileId>/`:

```
memory/
  packages/
    <logicalId>/
      manifest.json
      revisions/
        sha256-<hex>.json
        sha256-<hex>.json
      artifact-refs.json
      README.md
  artifacts/
    sha256/
      <shard>/<shard>/sha256-<hex>
indexes/
  records.db  (SQLite with records, revisions, edges, artifacts, temporal,
               reminders tables + records_fts FTS5 virtual table)
storage-format.json  ({version: 1, hashAlgorithm: "sha256", ...})
```

Each revision file is a pretty-printed JSON object — the canonical hash is
computed on the compact canonicalised form, but what is stored on disk is
human-readable.

## Related

- [provider.md](provider.md) — how the provider wraps this store
- [indexes.md](indexes.md) — the SQLite schema for fast lookup
- [../docs-ingestion.md](../docs-ingestion.md) — the ingestion pipeline that
  populates records