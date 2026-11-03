# Alfred Self-Knowledge Wiki

This directory is Alfred's guidebook to its own construction. Every document
here describes **how Alfred actually works** — the code paths, the data
structures, the files on disk, the lifecycle of a turn.

It exists because Alfred's persona files (`SOUL.md`, `IDENTITY.md`) are a
prompt prefix paid for on every single turn, and are therefore the wrong place
to keep detailed knowledge. This wiki is the large, navigable body of
self-knowledge that the small always-on introduction points toward.

## Scope

**In scope:** Alfred's own implementation. Package layout, the turn lifecycle,
the OIP memory substrate, retrieval mechanics, ingestion pipelines, the tool
surface, persona loading, the runtime surfaces.

**Out of scope (for now):** facts about the user, conversation content,
household data, project notes. Those belong in the memory graph, not here.
This wiki is strictly self-referential.

## How to read this

Each subdirectory groups documents by topic closeness and carries its own
`README.md` as the index for that area. Start with the area that matches your
question:

| Area | What it covers | Start here |
|---|---|---|
| [`identity/`](identity/README.md) | Persona files, how they load, the layered design, the HYPATIA override | [identity/README.md](identity/README.md) |
| [`memory/`](memory/README.md) | Memory providers, retrieval, ingestion pipelines, the graph | [memory/README.md](memory/README.md) |
| [`memory/oip/`](memory/oip/README.md) | OIP internals: records, revisions, hashing, indexes | [memory/oip/README.md](memory/oip/README.md) |
| [`runtime/`](runtime/README.md) | Turn lifecycle, prompt assembly, session orchestration, state machine | [runtime/README.md](runtime/README.md) |
| [`capabilities/`](capabilities/README.md) | The tool surface, agent router, harnesses, delegation | [capabilities/README.md](capabilities/README.md) |
| [`surfaces/`](surfaces/README.md) | Voice, text, desktop, bot — the ways Alfred is reached | [surfaces/README.md](surfaces/README.md) |
| [`operations/`](operations/README.md) | Repo layout, configuration, CLI commands, verification | [operations/README.md](operations/README.md) |

## Frequent questions, and where they are answered

| Question | Document |
|---|---|
| Where do my files live? | [operations/repository-layout.md](operations/repository-layout.md) |
| What happens when someone talks to me? | [runtime/turn-lifecycle.md](runtime/turn-lifecycle.md) |
| What states can a turn be in, and which transitions are legal? | [runtime/state-machine.md](runtime/state-machine.md) |
| How is my system prompt built? | [runtime/prompt-assembly.md](runtime/prompt-assembly.md) |
| How does Alfred decide to call a tool vs. delegate vs. answer? | [runtime/intent-layer.md](runtime/intent-layer.md) |
| Why is Alfred using a different model provider than usual? | [runtime/failover.md](runtime/failover.md) |
| How is a memory stored? | [memory/oip/record-model.md](memory/oip/record-model.md) |
| How do I find a memory? | [memory/retrieval.md](memory/retrieval.md) |
| Why do I sometimes miss a memory I should have? | [memory/retrieval.md](memory/retrieval.md) — see the no-embeddings finding |
| Where do people and facts in conversation become records? | [memory/conversation-extraction.md](memory/conversation-extraction.md) |
| Which record is "me"? | [memory/self-identity.md](memory/self-identity.md) |
| Where does my personality come from? | [identity/persona-files.md](identity/persona-files.md) |
| What tools do I have? | [capabilities/tool-surface.md](capabilities/tool-surface.md) |
| How does this wiki get into my memory? | [memory/docs-ingestion.md](memory/docs-ingestion.md) |
| How do I change and verify something? | [operations/change-and-verification.md](operations/change-and-verification.md) |

## Conventions used throughout

- **Code references are paths plus symbols**, e.g.
  `packages/core/src/session.ts:433`. Paths are relative to the repo root
  (`/home/controlroomtoon/alfred/`).
- **Headings are retrieval boundaries.** The ingestion chunker splits markdown
  at headings, so each `##` section becomes an independently retrievable
  record. Sections are written to stand alone.
- **Verified vs inferred.** Statements here were checked against the source.
  Where something is inferred, untested, or uncertain, it says so explicitly.
- **Cross-links are relative** so the wiki is traversable both on disk and
  after ingestion.

## Caveats

**This wiki is commits-only right now.** Alfred's current tool surface has no
`read_file` or `open_memory` tool, so no running session can open these files
directly, and no routing instruction in SOUL.md yet points at them. The wiki is
an authored corpus on disk; it is not reachable at runtime. See the Software
Craft project at `vault/projects/2026-11-02-software-craft/` — Project A covers
the tools that will make these files readable by a running Alfred session.

**The wiki may be stale.** Check the version marker below against the current
`origin/main` tip — if upstream has added significant new code, the wiki's
descriptions of the tool surface, memory system, and runtime may need updating.

## Version

Authoring commit: `cc7f7ec` on `feat/memory-graph-nodes` (2026-11-03)
Upstream at authoring: `b38643b`
Upstream merged: `6425b55` (2026-11-03) — 182 files, +20,615/-971
  Adds graph-report.ts (509 lines), audio-note-ingest (309 lines), x-ingest
  expansion, calculate/log_medicine/get_fitness_* tools, add_briefing_module,
  and note stills. **Not yet reflected in the wiki.**

To check drift against upstream:
```bash
git diff b38643b..origin/main --stat | head -15
```

To check drift since the wiki was authored:
```bash
git diff cc7f7ec..origin/main --stat | grep -v self-wiki/
```
