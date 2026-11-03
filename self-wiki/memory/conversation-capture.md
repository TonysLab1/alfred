# Conversation Capture — How Turns Become Memory

Every user utterance and every assistant reply is written into OIP as an
Observation. This document covers the `commitTurn` path in
`packages/memory/src/oip-local/provider.ts` and the related session orchestrator
code.

## The orchestrator's side

In `packages/core/src/session.ts`, `SessionOrchestrator` calls
`memory.commitTurn()` twice per normal turn — once for the user and once for
the assistant.

User turn (line 361):

```ts
await this.opts.memory.commitTurn({
  profileId: this.opts.profileId,
  sessionId: this.sessionId,
  turnId: turn.id,
  role: "user",
  text,
  metadata: {},
});
```

Assistant turn (line 433):

```ts
await this.opts.memory.commitTurn({
  profileId: this.opts.profileId,
  sessionId: this.sessionId,
  turnId: assistantTurn.id,
  role: "assistant",
  text: assistantTurn.text,
  metadata: {
    userText: text,
    toolcallCount: track.toolcallCount,
    reasoningLineCount: assistantTurn.text.split('\n').length,
  },
});
```

The assistant commit carries metadata that the user commit does not:
`userText`, `toolcallCount`, and `reasoningLineCount`. These are passed into the
`commitTurn` contract under `metadata` (a `Record<string, unknown>`).

## The OIP provider's side

`commitTurn()` in `packages/memory/src/oip-local/provider.ts:245` runs inside an
exclusive lock and does three things:

### 1) Create an Observation for the turn

A new Observation package is created with:

```ts
const observation = await this.packages.createPackage({
  type: "Observation",
  now,
  body: {
    text: commit.text,
    observedAt: now,
    schema: {
      "@type": "CreativeWork",
      text: commit.text,
      name: turnName,
    },
    schemaType: "https://schema.org/CreativeWork",
    alfred: {
      visibility: "private",
      confidence: 1,
      assertionType: "explicit",
      toolcallCount: commit.metadata?.toolcallCount ?? 0,
      reasoningLineCount: commit.metadata?.reasoningLineCount ?? 0,
    },
    provenance: {
      sourceType: "conversation_turn",
      learnedAt: now,
      speaker,                 // "user" or "assistant"
      extractionMethod: "commitTurn",
    },
    drefs: {
      session: commit.sessionId,
      turnData: `${commit.sessionId}/${commit.turnId}`,
    },
  },
});
```

Key properties of the resulting Observation:

- `drefs.session` points to the session
- `drefs.turnData` is a compound pointer `sessionId/turnId` — one targeted
  query against the turn store retrieves the full turn record including tool
  call data
- `alfred.toolcallCount` and `alfred.reasoningLineCount` are light metadata
  on the assistant turn — no tool call transcripts in the node prose
- `provenance.extractionMethod = "commitTurn"` — distinguishes this from
  document extraction (which is `"docs_heading_chunk"` or
  `"docs_folder_ingest"`)

### 2) Incrementally index the new observation

```ts
this.indexRevisionUnlocked(observation);
```

This indexes the new record into the SQLite tables without a full rebuild. The
provider avoids full-rebuild on every turn — the comment explicitly warns that
doing so races and drops memories.

### 3) Run conversational memory extraction on the text

```ts
const { extractConversationalMemory, writeConversationalMemory } =
  await import("../conversation-memory.js");
const extracted = extractConversationalMemory(commit.text);
const hasStructured = …;
if (hasStructured) {
  await writeConversationalMemory(this, extracted, { sessionId: commit.sessionId });
}
```

`extractConversationalMemory` is a regex/heuristic parser defined in
`packages/memory/src/conversation-memory.ts`. It extracts Person and
Organization entities, Assertions, and free-text durable notes from the turn
text using name-cleanup and plausibility checks — no LLM, just pattern matching.

If the extraction yields entities or assertions, `writeConversationalMemory`
writes them as structured OIP packages and wires them into the graph via the
relationships stored in drefs.

This means conversation capture is not passive transcription. It is an active
extraction pipeline that produces typed OIP records from every turn.

## What YOU can find from a stored turn

For a turn Observation retrieved via memory search, the content format displays
the observation text with metadata: `[sourceType=conversation_turn;
speaker=user|assistant; …]`.

For an assistant turn that involved tool use:

- `drefs.turnData` is the key (`sessionId/turnId`) — one query gets the full
  turn including tool call records
- `alfred.toolcallCount` tells whether tools were involved without loading the
  turn data
- the Observation's text contains the delivered assistant reply (or the
  committed text if nothing was delivered)

## Timelines

Both `createdAt` and `updatedAt` are set to `new Date().toISOString()` at the
moment of the commit. The user turn was spoken (and committed) before the
assistant turn, but both carry the same approximate timestamp because a normal
turn completes in well under a minute.

The Observation's `learnedAt` and `observedAt` fields are similarly stamped at
commit time, not at the time of original speech.

## Related

- [providers.md](providers.md) — the provider routing context
- [oip/record-model.md](oip/record-model.md) — the full OIP record schema
- [../runtime/turn-lifecycle.md](../runtime/turn-lifecycle.md) — where the two
  commitTurn calls occur in the turn sequence