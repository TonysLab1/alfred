# Persona Override — The HYPATIA Branch

Alfred already contains a complete persona-swap mechanism. It is not
configuration-driven and not documented anywhere in the persona files — it is a
code branch keyed to a single environment variable. Anyone reasoning about
"session masks" or alternate personas should start here, because the capability
already exists.

## The trigger

Setting `ALFRED_ARCHIVE_URL` to a non-empty value switches Alfred into HYPATIA,
the librarian of a local Alexandria archive.

```ts
const archiveUrl = process.env.ALFRED_ARCHIVE_URL?.trim();
const archive = archiveUrl ? createHttpArchivePort(archiveUrl) : undefined;
```

`apps/voice-agent/src/brain.ts:224`.

## What gets replaced

In the voice agent, the loaded persona is **discarded entirely** and rebuilt
from string constants:

```ts
const persona = archive
  ? {
      dir: personaLoaded.dir,
      soul: HYPATIA_SOUL,
      identity: HYPATIA_IDENTITY,
      user: "The user is researching documents in a local Alexandria archive.",
    }
  : personaLoaded;
if (archive) {
  config.systemInstructions = HYPATIA_INSTRUCTIONS;
}
```

`apps/voice-agent/src/brain.ts:226-238`.

Note `dir` is preserved while all three content fields are overridden — the
persona directory is still reported, but nothing from it reaches the prompt.
SOUL.md and IDENTITY.md on disk become dead weight for the session.

`systemInstructions` is also rewritten, so the override reaches past the
persona layer into the base instruction block. See
[../runtime/prompt-assembly.md](../runtime/prompt-assembly.md) for where each
lands.

## The constants

All three are defined at the top of `apps/voice-agent/src/brain.ts`:

- **`HYPATIA_INSTRUCTIONS`** (line 64) — behavioural routing: call
  `search_archive` or `ask_archive` for documents, people, places, events, and
  collections; name the records used; distinguish source documents from AI
  analyses; never state a truth score or percentage; keep spoken answers to a
  few sentences.
- **`HYPATIA_SOUL`** (line 70) — *"You sit beside the reader. You are concise,
  precise, and unwilling to flatten evidence into a score. If the archive does
  not contain something, say so. Possible name matches stay possible; do not
  merge them."*
- **`HYPATIA_IDENTITY`** (line 74) — *"HYPATIA — HYPermedia Agent for
  Transparent Information Access. You speak for the local Open Index, not for
  Alfred's personal assistant duties, while this archive is connected."*

## The desktop path does it differently

`apps/desktop-client/src/lib/text-session.ts:178` applies the same trigger but
only swaps `systemInstructions`, via a ternary:

```ts
systemInstructions: process.env.ALFRED_ARCHIVE_URL?.trim()
  ? "You are HYPATIA, the librarian of a local Alexandria archive. …"
  : "You are ALFRED, a helpful personal assistant. …",
```

The persona files themselves are loaded normally and still injected. So the two
surfaces disagree about what the override means:

| Surface | `systemInstructions` | SOUL / IDENTITY / USER |
|---|---|---|
| Voice agent (`brain.ts`) | Replaced with HYPATIA | **Replaced** with HYPATIA constants |
| Desktop text (`text-session.ts`) | Replaced with HYPATIA | **Loaded from disk as normal** |

In desktop mode with the archive connected, Alfred is told it is HYPATIA while
simultaneously being handed ALFRED's SOUL.md. This is an inconsistency between
surfaces, not a designed difference — recorded here as an observation, not as
intended behaviour.

## Tool surface also changes

The archive port adds archive-specific tools. `live/instructions.ts` includes a
conditional block:

```ts
...(brain.archive
  ? ["When the user asks about documents, people, collections, or the archive, call search_archive or ask_archive. Name the records. Never state a truth percentage."]
  : []),
```

and the backend tool list appends `search_archive`,
`open_archive_record`, `archive_neighbors`, `ask_archive` when `brain.archive`
is set. See [../capabilities/tool-surface.md](../capabilities/tool-surface.md).

## Why this matters for persona design

Three things follow from this branch existing:

1. **A whole-persona swap is already proven to work.** The plumbing —
   substituting the `PersonaContext` object before the session is constructed —
   is in place and exercised.
2. **It is code, not configuration.** Adding a second mask means adding another
   branch and another set of string constants. Masks are not currently
   authorable without touching TypeScript.
3. **It is invisible from the persona directory.** Nothing in SOUL.md,
   IDENTITY.md, or the loader hints that an env var can replace all of them. A
   reader inspecting only `data/persona/` would conclude persona comes solely
   from those files.

If persona variants ever become a real feature, generalising this branch into
configuration — a named mask directory selected at runtime — is the path that
reuses working machinery. Adding a fourth static persona file does not.

## Related

- [persona-files.md](persona-files.md) — the files this override bypasses
- [persona-loading.md](persona-loading.md) — the loader whose output is discarded
- [../runtime/prompt-assembly.md](../runtime/prompt-assembly.md) — the injection
  points affected
- [../capabilities/tool-surface.md](../capabilities/tool-surface.md) — archive
  tools added alongside
