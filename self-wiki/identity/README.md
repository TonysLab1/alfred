# Identity — Persona Files and How Alfred Knows Who It Is

Alfred's sense of self is not in its weights. It is read from markdown files on
disk at startup and injected into the system prompt on every turn. This area
documents that mechanism, the files involved, and the design tension built into
it.

## Documents in this area

| Document | What it covers |
|---|---|
| [persona-files.md](persona-files.md) | The three files — SOUL, IDENTITY, USER — what each holds and the seed-on-missing behaviour |
| [persona-loading.md](persona-loading.md) | The loader: path resolution, char budgets, frontmatter stripping, the caching gotcha |
| [user-model.md](user-model.md) | USER.md as machine-maintained state: the ingest patcher, the budget rebuilder, the dedupe pass |
| [persona-override.md](persona-override.md) | The HYPATIA branch — an existing whole-persona swap triggered by an env var |
| [layered-self-knowledge.md](layered-self-knowledge.md) | Why identity splits into introduction / wiki / graph, and what belongs in each |

## The short version

Three files live in `data/persona/<profileId>/`:

- **SOUL.md** — core identity and conduct. Human-written.
- **IDENTITY.md** — surface presentation: name, vibe, emoji. Human-written.
- **USER.md** — a model of the person Alfred serves. **Machine-written.**

They are loaded once per runtime by `loadPersonaContext()` in
`packages/memory/src/persona.ts` and pushed into the system prompt ahead of
retrieved memory by `PromptAssembler.assemble()`.

Two properties matter more than the contents:

1. **The set of files is closed.** `PERSONA_FILES` is a hardcoded three-element
   tuple and `PersonaContext` is a four-field schema. Adding a new markdown file
   to the persona directory has no effect — nothing reads it. See
   [persona-loading.md](persona-loading.md).
2. **Only USER.md is machine-written.** SOUL and IDENTITY are never modified by
   code. That is a real, enforced-by-construction boundary on self-modification.
   See [user-model.md](user-model.md).

## Why this area is small and the memory area is large

Everything in the persona directory is paid for on **every turn of every
session, forever**. The budgets reflect that: 20,000 chars for SOUL and
IDENTITY, 12,000 for USER. Those are ceilings, not targets.

This is the reason the self-wiki exists as a separate body of knowledge rather
than more persona files — see
[layered-self-knowledge.md](layered-self-knowledge.md) for the full argument.

## Related

- [../runtime/prompt-assembly.md](../runtime/prompt-assembly.md) — where persona
  text lands in the assembled prompt and in what order
- [../memory/docs-ingestion.md](../memory/docs-ingestion.md) — how this wiki
  becomes retrievable memory instead of prompt prefix
