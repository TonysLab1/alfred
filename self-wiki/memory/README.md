# Memory — Providers, Retrieval, and What Is Actually Stored

Alfred has two memory providers registered in every session, but the OIP
provider is the one that matters for structured knowledge. This area covers
what is stored, how retrieval finds it, and how content enters the graph.

## Documents in this area

| Document | What it covers |
|---|---|
| [providers.md](providers.md) | The two providers (JSONL and OIP), the controller, and which one is active |
| [conversation-capture.md](conversation-capture.md) | How `commitTurn` turns user and assistant utterances into OIP Observations |
| [conversation-extraction.md](conversation-extraction.md) | The regex pipeline that extracts people, orgs, and assertions from each turn |
| [self-identity.md](self-identity.md) | How the system decides which Entity record is "me" |
| [extraction-contract.md](extraction-contract.md) | The Zod schema governing structured LLM extraction output |
| [retrieval.md](retrieval.md) | The `retrieveMemories` function: FTS, name match, graph expansion — and what is missing |
| [docs-ingestion.md](docs-ingestion.md) | The `ingestDocsFolders` pipeline: walk, chunk, extract, write, ledger |
| [oip/README.md](oip/README.md) | OIP internals: record model, revision chain, package store, index schema |

## The short version

Every session has two providers behind a `MemoryController`:

- `LocalFileMemoryProvider` — a flat JSONL file on disk. Fast for keyword scan
  but no structure.
- `OipLocalMemoryProvider` — packages with DIDs, revision chains, typed drefs,
  SQLite/FTS/graph indexes, and artifact storage.

Structured writes (entities, assertions, extracted knowledge) always go to OIP,
regardless of which provider is active for conversational recall.

Retrieval is hybrid FTS + name match + graph expansion, with no vector step in
the current path. The vector index exists as an interface but the default
implementation (`NoopVectorIndex`) returns empty — embeddings are optional and
not yet wired in.

## The most important finding about retrieval

`retrieveMemories()` in `packages/memory/src/oip-local/retrieval.ts` does:

1. relationship recall heuristics (structured employer/boss/birthday queries)
2. FTS5 lexical search via SQLite, with bm25 scoring
3. proper-name extraction and entity-name lookup
4. graph BFS expansion from seeds, up to 2 hops
5. source-type boosting: docs, X.com, YouTube, audio notes, photo uploads

It does **not** run any kind of embedding / vector similarity search. The
`VectorIndex` is a pluggable interface defaulting to `NoopVectorIndex` —
`search()` always returns `[]`. So "semantic search" and "nearest-neighbor
retrieval" are present as empty interfaces but functionally absent in the active
retrieval path.

See [retrieval.md](retrieval.md) for the full chain.

## Related

- [../runtime/turn-lifecycle.md](../runtime/turn-lifecycle.md) — where retrieve
  and commitTurn are called
- [../runtime/prompt-assembly.md](../runtime/prompt-assembly.md) — the "Memory
  access model" block that conditions retrieval expectations
- [../identity/layered-self-knowledge.md](../identity/layered-self-knowledge.md)
  — why some knowledge belongs here rather than in persona files
- [../operations/memory-cli.md](../operations/memory-cli.md) — the CLI surface
  for docs ingestion and maintenance