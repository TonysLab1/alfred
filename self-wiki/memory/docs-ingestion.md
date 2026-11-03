# Document Ingestion — How Markdown Folders Become Retrievable Memory

The pipeline at `packages/memory/src/docs-ingest/` walks a markdown directory,
splits files into heading sections, optionally extracts structured facts via
LLM, writes OIP records, and keeps a content-hash ledger so re-runs are
incremental.

This document covers the pipeline that this self-wiki will ride on.

## The single entry point

`ingestDocsFolders(opts)` in `packages/memory/src/docs-ingest/ingest.ts` is the
orchestrator. It is called from the memory CLI:

```bash
pnpm memory -- ingest-docs [--path /abs/path] [--dry-run]
```

If no `--path` is given, it reads all registered sources from
`data/memory/<profile>/ingest/docs-sources.json`. Sources are registered with:

```bash
pnpm memory -- ingest-docs-source add --path /abs/path [--label "label"]
pnpm memory -- ingest-docs-source list
pnpm memory -- ingest-docs-source remove --path /abs/path
```

## The pipeline steps

### 1 — Walk

`walkMarkdownFiles(root)` in `walk.ts` recursively collects `.md`, `.txt`,
`.rtf`, and `.pdf` files. It skips directories named `.git`, `node_modules`,
`dist`, or `data` (and any starting with `.`).

**This is the skip-list gotcha.** If the self-wiki is placed under a `data/`
directory anywhere in the path, the walker silently returns zero files. No
error, no warning — `ingestDocsFolders` reports an empty run that looks like
success. This is why the wiki lives at repo root rather than inside `data/`.

### 2 — Chunk

`chunkMarkdown(markdown, relPath)` in `chunk.ts` splits the markdown by heading
level. Each heading becomes a `DocsChunk` with a key derived from the slugged
title. Oversized sections (over 3,500 chars) are further split on blank lines,
never inside fenced code blocks.

Headings are the retrieval grain. A page with three `##` sections becomes three
separately retrievable Observations. Write to this grain deliberately.

### 3 — Extract

`defaultDocsExtractor(input)` in `extract.ts` calls an LLM if
`OPENAI_API_KEY` is set, requesting JSON output against the extraction contract.
The prompt is explicit: "Only state facts that appear in the text. Do not
invent architecture, APIs, or behavior."

If the key is absent, `noopDocsExtractor` returns empty extraction arrays and
only the verbatim section Observations are written. This degraded mode is still
useful — the FTS index covers verbatim text well.

The extraction contract:
```ts
MemoryExtractionResultSchema = {
  entities, episodes, observations, assertions, relationships,
  temporalReferences, scheduledSurfacing, publicKnowledge,
  memoryImportance, ambiguities, needsResolution
}
```

### 4 — Write to OIP

`writeDocsFileToOip()` in `oip-write.ts` produces for each ingested file:

- one **Artifact** (the raw bytes, content-addressed)
- one **DigitalDocument Entity** (the file) with `drefs.isPartOf` pointing at
  the folder Collection
- one **Observation** per chunk section (the verbatim text), also linked to
  the file and folder
- **Extracted records** (Entities, Assertions, etc.) from the LLM extraction,
  linked to the section Observation via drefs

### 5 — A folder Collection entity is created

`upsertFolderEntity()` creates or updates a `Collection` entity per source
label, with `alfred.entityClass = "docs_folder"`. All files and sections link
back to this entity, so the retrieval source-type booster can find them.

### 6 — Superseded sections are cleared

If a re-ingest finds that a section heading that existed in the previous run is
now absent from the file, it writes a superseding observation with empty text
and `supersedes: [oldId]`. So stale records are not rotting in the index — they
are explicitly marked as gone.

### 7 — Extracted records from a previous run are deleted

If extraction was run before and new extraction replaces old extracted records,
the previous extract DIDs are deleted from the package store. The verbatim
observations are updated in place; the extracted records are replaced wholesale
per file.

## The ledger — what makes this idempotent

`docs-ledger.jsonl` at `data/memory/<profile>/ingest/docs-ledger.jsonl` stores
one JSON line per ingested file:

```json
{"path":"/abs/path/to/file.md","relPath":"...","contentHash":"sha256:...",
 "folderDid":"did:memory:...","fileDid":"did:memory:...","sectionKeys":{...},
 "extractDids":[...],"lastIngestedAt":"2026-..."}
```

On re-ingest, if a file's content hash matches the ledger, it is skipped with
status `"skipped"`. If the hash changed (edit) or the file is new, it is
re-ingested.

This means the wiki is a living document: edit a page, re-run `pnpm memory --
ingest-docs`, and only that page's records are rewritten. No full rebuild.

## The SKIP_DIRS problem in detail

From `packages/memory/src/docs-ingest/walk.ts`:

```ts
const SKIP_DIRS = new Set([".git", "node_modules", "dist", "data"]);
```

and:

```ts
if (SKIP_DIRS.has(entry.name) || entry.name.startsWith(".")) continue;
```

This checks the **directory name**, not the position relative to the walk root.
So any directory named `data` anywhere in the walk path causes its subtree to be
silently skipped.

For the self-wiki, the solution is to place it outside `data/` — at repo root,
e.g. `/home/controlroomtoon/alfred/self-wiki/`. This also puts it under git
tracking, which `data/` is not (`.gitignore` line 15: `**/data/`).

This behaviour is arguably a bug for any folder-watching use case where data
lives under the repo's data directory. A fix matching against the walk root
prefix would be more correct, but is a separate change with its own blast
radius.

## Current state of this pipeline on this machine

The ledger at `data/memory/profile.default/ingest/docs-ledger.jsonl` is 0 lines
— confirmed 2026-11-02. No docs folder has ever been ingested.

`docs-sources.json` is similarly absent or empty.

The pipeline is implemented, tested, and wired to the CLI — it has simply never
been run.

## Verification

After running `pnpm memory -- ingest-docs --path ./self-wiki`:

- `docs-ledger.jsonl` should be non-empty
- a `Collection` entity named "Alfred Self-Knowledge" (or whatever `--label` is
  passed) should exist in OIP
- per-file `DigitalDocument` records and per-section `Observation` records
  should exist
- `search_memory("turn lifecycle")` should return wiki content
- a re-run with no edits should report everything as `[skipped]`

## Related

- [retrieval.md](retrieval.md) — what happens when the ingested content is
  searched
- [oip/record-model.md](oip/record-model.md) — what records look like
- [../operations/memory-cli.md](../operations/memory-cli.md) — the full CLI
  command surface