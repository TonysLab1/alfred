# Prompt Assembly — What Goes Into Alfred's System Prompt

The prompt assembler lives at `packages/core/src/prompt-assembler.ts`. It takes
structured input and produces an `AssembledPrompt` rather than blindly
concatenating text.

This document covers the assembly order and the unconditional guidance blocks,
because both are part of Alfred's runtime identity.

## The order matters

`PromptAssembler.assemble()` builds `systemParts` beginning with
`input.systemInstructions.trim()`, then appends additional blocks in a fixed
sequence.

The sequence, as implemented:

1. `systemInstructions`
2. `SOUL.md (persona / boundaries)` if present
3. `IDENTITY.md (who you are)` if present
4. `USER.md (user model)` if present
5. `extraSystem` if present
6. capability guidance derived from `availableCapabilities`
7. an unconditional **Memory access model** block
8. due reminders, if any
9. retrieved long-term memory
10. the current user turn and recent conversation context (in the returned
    message structure)

The corresponding note flags are appended alongside, e.g. `soul_attached`,
`identity_attached`, `user_model_attached`, `extra_system_attached`,
`memory_access_model`, `due_reminders_attached`.

`prompt-assembler.test.ts` explicitly asserts that persona blocks appear before
retrieved memory.

## Persona injection is simple string insertion

For each persona field, the assembler pushes a labelled block:

```ts
if (persona?.soul) {
  notes.push("soul_attached");
  systemParts.push(`SOUL.md (persona / boundaries):\n${persona.soul}`);
}
```

The same pattern is repeated for identity and user. There is no parsing of the
markdown structure; the body is injected as raw text after frontmatter stripping
and truncation in the loader.

See [../identity/persona-loading.md](../identity/persona-loading.md).

## Capability guidance is generated from the available tool surface

After persona and extra system text, the assembler takes
`availableCapabilities`, turns it into a set, and conditionally appends tool-use
guidance.

Examples currently handled in `prompt-assembler.ts`:

- `delegate_task` — use delegate_task for external harness actions instead of
  inventing harness-specific tools
- `update_reminder` — when a reminder is done / snoozed / should stop, call the
  tool rather than improvising
- `remember_memory` — lasting people/fact/preference information must be stored
  with the tool, not just acknowledged verbally
- `get_weather_forecast`, `get_news_headlines`, `summarize_news_article`,
  `get_crypto_price`, `get_metals_price`, `get_current_time`,
  `get_earthquakes`, `get_weather_alerts`, `get_space_weather`,
  `get_natural_events`, `get_exchange_rate`, `get_hacker_news`,
  `control_studio_lights`

This block is how Alfred learns tool-routing habits from the runtime without
having every tool's rules hardcoded into SOUL.md.

## The memory access model block is always present

Regardless of capabilities or persona, the assembler pushes this guidance:

- ingested documents, docs folders, knowledge exports, audio notes, and X notes
  are extracted into long-term memory graph storage
- Alfred does **not** open those files on disk during conversation
- when the user asks about something they ingested or "in a document/file",
  Alfred must treat it as a memory question
- Alfred must never claim it lacks access to ingested material; retrieval is how
  it reads it

This is a crucial runtime invariant because it corrects a common failure mode:
confusing "I do not have a generic file-reading tool in this context" with
"I cannot read ingested material at all."

It is also the direct precedent for telling Alfred that self-wiki content, once
ingested, should be reached via retrieval rather than via disk access.

## Due reminders are rendered into a numbered block

When due reminders are present, the assembler renders one line per reminder:

```ts
[1] id=<recordId> remindAt=<ts> status=<status> — <summary>
```

and prefixes them with a sentence reminding Alfred to call `update_reminder`
when the user indicates done / stop / snooze.

So reminders are not just tool-available; they are actively surfaced in prompt
text on turns where they matter.

## Retrieved memory is attached after persona and operational guidance

The persona and routing guidance come first, the retrieved memory later. The
stated test intent in `prompt-assembler.test.ts` is exactly that: *"injects
SOUL / IDENTITY / USER before retrieved memory"*.

This means the retrieved memory is always interpreted under the frame supplied by
persona + capability guidance + the memory access model, rather than the other
way around.

## GPT-Live has a parallel assembly path

The voice live stack uses `apps/voice-agent/src/live/instructions.ts`, not
`PromptAssembler`, but the pattern is recognisably parallel.

`buildLiveVoiceInstructions(brain)` builds a top-level instruction string with:

- `brain.config.systemInstructions`
- GPT-Live-specific spoken interaction rules
- archive-specific rules when `brain.archive` exists
- reminder / briefing / news / time / weather / market / lights / memory
  delegation guidance
- then `SOUL.md`, `IDENTITY.md`, `USER.md`

`buildLiveBackendInstructions(brain)` separately constructs a backend tool-use
instruction block listing the actual callable tools.

So there are two assembly systems:

| Path | File | Purpose |
|---|---|---|
| Standard text / orchestrator path | `packages/core/src/prompt-assembler.ts` | normal structured conversation prompt |
| GPT-Live voice path | `apps/voice-agent/src/live/instructions.ts` | top-level spoken instructions + backend tool-use instructions |

They are conceptually aligned but not literally the same code, which means they
can drift.

## Consequences for self-knowledge design

### The persona layer should stay small
Everything in SOUL / IDENTITY / USER lands before retrieved memory, on every
turn. This makes those files the most expensive possible place to keep detailed
knowledge.

### Routing guidance belongs here
The correct always-on instruction is not "remember every implementation detail"
but "when the topic is your own construction, retrieve self-knowledge from
memory." The memory access model block already proves this pattern exists for
other ingested material.

### Surface drift is real
Because GPT-Live has its own assembly path, any change to how Alfred is taught
to use self-knowledge has to consider both files, not just `PromptAssembler`.

## Related

- [turn-lifecycle.md](turn-lifecycle.md) — where assembly is called
- [../identity/persona-loading.md](../identity/persona-loading.md) — where the
  fields come from
- [../memory/docs-ingestion.md](../memory/docs-ingestion.md) — how documents
  become retrievable memory
