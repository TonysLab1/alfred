# Layered Self-Knowledge — Why This Wiki Exists

The design argument behind splitting Alfred's self-knowledge into three layers,
and the rule for deciding which layer a given piece of knowledge belongs in.

## The three layers

The split is by **access pattern**, not by subject matter.

| Layer | Holds | Size | Mechanism | Written by |
|---|---|---|---|---|
| **Introduction** | Who I am, how I behave, that a self-wiki exists and how to reach it | Small | Prompt prefix, every turn | Human only |
| **Self-wiki** | How I actually work: layout, lifecycle, tools, mechanisms | Unbounded | Markdown → ingested → retrieved on relevance | Human, version-controlled |
| **Graph** | Facts extracted from the wiki and from conversation | Unbounded | OIP records, relevance-ranked retrieval | Machine, via extraction |

Layer 1 is the persona directory
([persona-files.md](persona-files.md)). Layer 2 is this wiki. Layer 3 is OIP
([../memory/oip/README.md](../memory/oip/README.md)).

## The rule

> **A prompt prefix is good at introduction and terrible at knowledge.**

A prefix is always present, frames everything, and is cheap when small. Those
are exactly the properties an introduction needs.

A prefix also cannot be large, cannot be queried, cannot be versioned
per-fact, and cannot be superseded by new evidence. Those are exactly the
properties knowledge needs.

So: **if it must be true on every turn regardless of topic, it goes in the
introduction. If it is only relevant when a particular subject comes up, it
goes in the wiki and the graph.**

## What goes where, concretely

| Knowledge | Layer | Why |
|---|---|---|
| "Be direct; skip performative helpfulness" | Introduction | Must hold on every turn |
| "Detailed self-knowledge is in memory — search it" | Introduction | Routing instruction; useless if not always present |
| "Session state machine has states X, Y, Z" | Wiki | Only relevant when the lifecycle is the topic |
| "Retrieval has no vector step" | Wiki | Only relevant when retrieval is the topic |
| "The user prefers short spoken answers" | Graph | A fact, subject to change, should be superseded not edited |

## The failure mode this avoids

USER.md is the worked example of getting it wrong, and it is in this codebase
right now.

Facts about the user were put in the always-on layer. Because the always-on
layer has a hard size limit, keeping it viable required: a marker-block
splicer, a two-threshold character budget, overflow demotion into memory
records, a hand-maintained junk regex, and a near-duplicate collapser.

Every one of those is a reimplementation of something OIP already does — and
does better, because OIP has stable ids, revision chains, and `supersedes` /
`contradicts` / `reinforces` as first-class fields.

Full detail: [user-model.md](user-model.md).

The generalisable lesson: **when a document needs an eviction policy, the
content is in the wrong layer.**

## Why the wiki does not need to agree with the persona files

Retrieval is ranked by relevance to the current query, not by global
consistency. A wiki page about the turn lifecycle surfaces when the turn
lifecycle is the topic; it does not have to be reconciled with SOUL.md's
framing of anything.

Where a genuine contradiction exists, OIP represents it directly — `contradicts`
is a first-class dref on every revision
([../memory/oip/record-model.md](../memory/oip/record-model.md)). A
contradiction is information, not a defect to be edited away.

This removes the coordination cost that would otherwise make a wiki expensive:
pages can be written independently, at different times, at different levels of
confidence, without a reconciliation pass across the whole corpus.

## Why not just add more persona files

Because the loader will not read them. `PERSONA_FILES` is a hardcoded
three-element tuple and `PersonaContext` is a closed four-field schema; a
fourth markdown file in the persona directory is inert. See
[persona-loading.md](persona-loading.md).

And even if the schema were extended, the cost model does not change: anything
in the persona directory is paid for on every turn of every session,
permanently. A wiki ingested into the graph is paid for only when retrieved.

## What makes the wiki reachable

Two things, and both are required:

1. **Ingestion** — the wiki must be compiled into retrievable records. Alfred
   already has the pipeline for this; see
   [../memory/docs-ingestion.md](../memory/docs-ingestion.md).
2. **A routing instruction in the introduction** — Alfred must know to search
   rather than guess or claim ignorance.

Without (2), the wiki is ingested and never consulted. There is precedent for
(2) already in the prompt: the assembler unconditionally injects a "Memory
access model" block stating that ingested documents live in memory, must be
retrieved, and must never be described as inaccessible. See
[../runtime/prompt-assembly.md](../runtime/prompt-assembly.md).

## Status of this design

The layering is **described, not fully implemented.** As of 2026-11-02:

- Layer 2 exists as files — this wiki.
- Layer 2's ingestion into Layer 3 has **not yet been run.**
- The introduction layer has **not** been rewritten; the persona files are
  still the shipped defaults, and carry no routing instruction toward the wiki.

Nothing here should be read as a description of currently working behaviour.
See [../operations/change-and-verification.md](../operations/change-and-verification.md)
for what would have to be verified.

## Related

- [persona-files.md](persona-files.md) — the introduction layer as it stands
- [user-model.md](user-model.md) — the counter-example in detail
- [../memory/docs-ingestion.md](../memory/docs-ingestion.md) — how layer 2
  reaches layer 3
- [../memory/retrieval.md](../memory/retrieval.md) — what "surfaces on
  relevance" actually means mechanically, including its current limits
