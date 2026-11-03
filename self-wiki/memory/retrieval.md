# Retrieval — How Alfred Finds Memories

The retrieval function is `retrieveMemories()` in
`packages/memory/src/oip-local/retrieval.ts`. It is called from
`OipLocalMemoryProvider.retrieve()`, which is called from
`MemoryController.retrieve()`, which is called from the session orchestrator
at `session.ts:352`.

## Retrieval type

The contract is `MemoryRetrievalResult`, which wraps an array of
`NormalizedMemoryItem`:

```ts
interface NormalizedMemoryItem {
  id: string;
  content: string;   // formatted prose representation
  sourceId: string;
  providerId: string;
  createdAt: string;
  updatedAt: string;
  confidence?: number;
  relevance: number;  // 0–1 score
  provenance: { kind, revision, type, schemaType?, ... };
}
```

## The hybrid retrieval chain

`retrieveMemories()` builds a scored map of record ids through a sequence of
independent scoring passes. Each pass adds to or bumps scores in the map; at the
end, the top `limit` (default 8) by score are resolved into full records and
returned.

### Pass 1 — relationship recall

`relationshipRecallHits()` detects structured-query patterns like "who is
X's boss," "X's birthday," "who works at X" and generates direct edge
lookups. These score at a higher base than lexical hits (0.9 range) so
that structured facts outrank mere mention.

```ts
for (const hit of await relationshipRecallHits(query.text, deps)) {
  bump(hit.id, hit.score);
}
```

### Pass 2 — lexical (FTS5)

`FtsIndex.search()` queries the SQLite FTS5 table with tokenised OR terms,
weights via bm25, and inverts rank into a score:

```ts
for (const hit of fts.search(query.text, limit * 3)) {
  bump(hit.recordId, 0.4 + hit.score * 0.4);
}
```

The `toFtsQuery()` tokeniser in `sqlite-index.ts:700` splits on non-alphanumeric
characters, lowercases, strips stopwords, and wraps each token in double-quotes
joined with `OR`.

Word forms (plurality, tense) are not stemmed or lemmatised — the query is exact
token match on the FTS5 index.

### Pass 3 — proper-name seed extraction

`extractProperNames()` pulls capitalised words (excluding question words) and
possessive forms. These become entity name lookups:

```ts
for (const name of nameTokens) {
  for (const row of deps.sqlite.findByName(name)) {
    bump(row.id, 0.75);
    seedIds.push(row.id);
  }
}
```

### Pass 4 — graph expansion from seeds

Top lexical hits and name-match hits become seeds for a BFS graph expansion up
to 2 hops:

```ts
if (seedIds.length) {
  const walk = graph.expand(seedIds, 2);
  for (const nodeId of walk.nodeIds) {
    bump(nodeId, (scores.get(nodeId) ?? 0.2) + 0.15);
  }
}
```

The `GraphIndex` in `indexes/graph-index.ts` walks both outbound and inbound
edges. The 2-hop expansion means: given seed A, include records one edge away
from A, and records one edge away from those. For dense graphs this can pull in
many nodes.

### Pass 5 — episodic product boosting

If the query looks like a "wine at Sarah's" / "restaurant we went to" style
question, the expansion edges are further boosted where they connect to Episodes
or products.

### Pass 6 — source-type boosting

Specific query patterns boost records by provenance source type. These are
pattern-matched against the query text, not the record content:

| Pattern | Boost | Source type boosted |
|---|---|---|
| X.com / twitter / x post | +0.25 | `x_com` |
| YouTube / video / channel | +0.25 | `youtube` |
| docs / documentation / architecture | +0.3 | `docs_folder` |
| pdf / uploaded document | +0.3 | `document_upload` |
| photo / picture / image / OCR | +0.3 | `photo_upload` |
| audio note / meeting note / transcript | +0.3 | `audio_note` |

### Pass 7 — temporal window

If the query contains a learned-at or published-at time window (e.g. "from last
week"), records in that range get a flat +0.35 boost.

### Pass 8 — structured fallback

A hardcoded `/\bwine\b/i` test runs an entity-type scan for wine-related entity
names. This is the only hardcoded-topic heuristic.

## What is NOT happening — the vector gap

The `VectorIndex` interface (`indexes/vector-index.ts`) defines `upsert`,
`search`, `rebuild`, and `clear`. `NoopVectorIndex` implements all four as
no-ops.

`retrieveMemories()` calls none of them. The vector index is not consulted in
any scoring pass.

The `FileVectorIndex` in `indexes/file-vector-index.ts` (245 lines) exists as
an implementation, and `pca.ts` contains PCA projection code, but neither is
wired into the retrieval path.

This means **semantic similarity is not a factor in retrieval.** When Alfred
responds with "I remember X," it is doing lexical match + entity-name lookup +
graph walk — not embedding proximity.

### Consequences for self-knowledge

For a self-knowledge wiki ingested as `docs_folder` records, retrieval
questions like "how does the turn lifecycle work" will match:

- FTS5: tokens `turn` and `lifecycle` against the wiki section content
- source-type boosting: the question matches `/documentation|architecture/`
  which triggers the `docs_folder` +0.3 boost
- graph expansion: if wiki records link to each other via drefs, hops may
  surface related pages

They will not be matched by semantic proximity to other questions about runtime
internals that use different vocabulary.

### What would make retrieval miss a self-knowledge question

Observable failure modes:

- the query uses words that do not appear in the wiki text (no FTS match, and no
  vector fallback)
- the query is phrased as a question that triggers a different source-type boost
  ("why do you act like X" — probably not caught as `docs_folder`)
- a wiki page section is over 3,500 chars and the relevant detail is in a part
  the chunker sliced into a separate record whose title/key doesn't match

## Content formatting

`toNormalized()` in `retrieval.ts` calls `formatContent()` which renders
records as:

- **Assertions:** `subject predicate object [metadata]`
- **Observations:** the `text` field verbatim, with metadata suffix
- **Entities with contact info:** `Person: Name — email …; phone … [metadata]`
- **Entities without:** `Entity: name [metadata]`
- **Fallback:** `type id [metadata]`

The `docs_folder` source type adds `source=docs` plus `file=<relPath>` and
`folder=<folderLabel>` to the metadata suffix.

## Retrieval is called per-turn with limit 5

In the orchestrator (`session.ts:348`):

```ts
const memory = await this.opts.memory.retrieve({
  text,
  profileId: this.opts.profileId,
  sessionId: this.sessionId,
  limit: 5,
});
```

A limit of 5 means at most five NormalizedMemoryItems make it into the prompt
in a normal conversation turn. That constraint means wiki content competes with
all other stored memory for retrieval slots — the wiki is not a separate index.

## Related

- [docs-ingestion.md](docs-ingestion.md) — how wiki content enters the index
- [oip/indexes.md](oip/indexes.md) — the SQLite tables behind FTS and graph
- [../runtime/turn-lifecycle.md](../runtime/turn-lifecycle.md) — where the
  retrieve call sits in the turn sequence