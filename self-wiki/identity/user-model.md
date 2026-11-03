# The User Model — USER.md as Machine-Maintained State

USER.md is the only persona file that code writes. It holds facts about the
person Alfred serves, and an entire subsystem exists to keep it inside a
character budget. This document covers that machinery and why its existence is
evidence of a design problem.

For the file's declared format see
[persona-files.md](persona-files.md); for the budget constant see
[persona-loading.md](persona-loading.md).

## The declared format

The template prescribes directives, one per entry, each with a metadata
comment:

```md
<!-- observed: YYYY-MM-DD | status: active -->

- Prefer concise spoken answers.
```

Rules from the template itself:

- begin each directive with an imperative — `Always`, `Never`, or `Prefer`
- record the observation date and either `active` or `superseded`
- when a preference changes, mark the old entry `superseded` and rewrite the
  active directive; **never append a contradictory active directive**
- keep stable communication style, relationships, and active-project context
- *"Episodic turns and extracted facts also live in the JSONL memory store;
  keep this file curated."*

That last line is the tell: the template acknowledges the same information
lives in the memory store, and asks for manual curation to keep the two from
diverging.

## Write path 1 — ingest patching

`writeUserPatch()` in `packages/memory/src/knowledge-ingest.ts:123` splices
content into the file. It ensures persona files exist, reads the current
contents (falling back to a bare `# USER.md — User Model\n` header if the read
fails), merges, and writes back.

Merging is marker-delimited. `cleanup-user-md.ts` carries the recognising
patterns:

```ts
const INGEST_BLOCK_RE =
  /<!--\s*alfred:ingest-export:([^:]+):start\s*-->([\s\S]*?)<!--\s*alfred:ingest-export:\1:end\s*-->/gi;
const LEGACY_INGEST_RE =
  /<!--\s*alfred:ingest-export:start\s*-->([\s\S]*?)<!--\s*alfred:ingest-export:end\s*-->/gi;
```

The back-reference `\1` means each ingest source owns a uniquely-keyed block
that can be replaced in place on re-ingest. The legacy pattern is an unkeyed
earlier format still recognised for collapsing.

Knowledge export ingestion routes high-priority items, how-to-work guidance,
and negative preferences here — per the comment at `knowledge-ingest.ts:73`.

## Write path 2 — the budget rebuilder

`cleanupUserMd()` in `packages/memory/src/cleanup-user-md.ts` rebuilds the file
to fit. Its targets derive from the loader's budget:

```ts
const targetChars = opts.targetChars ?? Math.floor(USER_MD_MAX_CHARS * 0.75);
const maxChars   = opts.maxChars    ?? Math.floor(USER_MD_MAX_CHARS * 0.92);
```

With `USER_MD_MAX_CHARS = 12_000`, that is a soft target around 9,000 chars and
a hard ceiling around 11,040.

What the pass does, per its own docstring:

- strips chat-log and ephemeral junk
- collapses multiple ingest blocks
- keeps identity, operating rules, and compressed persistent context
- **parks remaining detail as overflow memory notes**

The result type carries `overflowNotes: CanonicalMemoryRecord[]` — content that
didn't fit is converted into memory records. The budget is enforced by
demoting prose into the graph.

Junk removal is a hand-maintained regex blocklist:

```ts
const JUNK_LINE =
  /(\[\d{1,2}\/\d{1,2}\/\d{2,4}[^\]]*\]\s*Alfred:)|(password\s*[:=]\s*\S+)|…/i;
```

The alternatives include a chat-transcript line pattern, a password assignment
pattern, and a series of install-specific strings (`assistant turn failed`,
`wave choreography finalized`, `pca9685 board arrived`, `board confirmed
fried`, `akashml model list updated`, `daily backup verified`, `events refresh
logged`). That list is a record of specific junk observed in one real file and
patched out by hand.

## Write path 3 — deduplication

`packages/memory/src/dedupe-user-md.ts` removes near-duplicate bullets and
paragraphs while preserving structure, exposing `dedupeUserMd`, `isRedundant`,
and `normalizeForDedupe`. `cleanup-user-md.ts` imports all three.

## Why this machinery exists

Four mechanisms — marker-block splicing, a two-threshold size budget with
overflow demotion, a hand-maintained junk blocklist, and a near-duplicate
collapser — all serve one goal: keep an accumulating body of facts small enough
to sit in a prompt prefix.

Every one of those problems is already solved, better, by the system sitting
next to it. OIP provides stable identity (`did:memory:` + ULID), revision
chains, explicit supersession (`supersedes`, `contradicts`, `reinforces`),
content hashing, and relevance-ranked retrieval. See
[../memory/oip/record-model.md](../memory/oip/record-model.md).

Put plainly: **USER.md is a cache with an eviction policy wearing the costume
of a document.** The budget arithmetic, junk regex, dedupe pass, and overflow
demotion exist only because facts are being stored in the always-on layer
instead of the retrievable one.

The template's own "keep this file curated" instruction and its warning against
contradictory active directives are both manual reimplementations of
supersession — a thing OIP does natively as a first-class dref.

This is the clearest in-codebase evidence for the layering argument in
[layered-self-knowledge.md](layered-self-knowledge.md).

## Maintenance commands

Both passes are exposed through the memory CLI:

```bash
pnpm memory -- dedupe-user
pnpm memory -- cleanup-user
```

See [../operations/memory-cli.md](../operations/memory-cli.md) for the full
command surface.

## Related

- [persona-files.md](persona-files.md) — the file's declared purpose and format
- [layered-self-knowledge.md](layered-self-knowledge.md) — the design argument
  this document supplies evidence for
- [../memory/oip/record-model.md](../memory/oip/record-model.md) — what the
  graph already provides
