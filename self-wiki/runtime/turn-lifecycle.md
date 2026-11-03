# Turn Lifecycle — From User Input to Stored Assistant Reply

This document follows the main text-turn path through
`packages/core/src/session.ts`, using the code as it exists on branch
`feat/memory-graph-nodes`.

## The high-level flow

A normal turn in `SessionOrchestrator` runs in this order:

1. transcribe input into text
2. commit the user turn to persistence
3. retrieve long-term memory for the user's text
4. commit the user turn into OIP memory
5. short-circuit into delegation if the text matches a delegation or ingest task
6. assemble the prompt
7. generate an assistant reply, possibly including tool calls
8. commit the first-generation reply into the response ledger
9. process any addendum segments that arrive after tool calls
10. speak / deliver the committed text
11. persist the assistant turn
12. commit the assistant turn into OIP memory with metadata
13. transition the state machine to Listening or Idle

The rest of this document expands those steps.

## 1) User text enters, gets transcribed, and is persisted

At `session.ts:340+`, the turn begins with:

```ts
const sttResult = await this.transcribe(input);
const text = sttResult.text || input.text;
const turn = await this.commitUserTurn(text);
```

So even on text paths, the same method shape is used: there is always an input
object, transcription is always attempted, and the fallback is the raw text.

`commitUserTurn(text)` persists the conversational turn through the configured
turn repository before any retrieval or generation happens.

## 2) Memory retrieval happens before the user turn is written into OIP

Immediately after the user turn is persisted, the orchestrator asks memory for
relevant items:

```ts
const memory = await this.opts.memory.retrieve({
  text,
  profileId: this.opts.profileId,
  sessionId: this.sessionId,
  limit: 5,
});
```

This ordering matters:

- retrieval is based on the user text only
- the just-spoken user turn is **not yet** in the OIP memory graph when the
  search runs
- the user's utterance is only committed to OIP afterward

This avoids a trivial self-hit where the newly-written user turn would always be
retrieved on the same turn.

A `memory.retrieved` event is emitted with the item count and provider id.

## 3) The user turn is then committed into OIP as an Observation

After retrieval, the user turn is written to the memory provider:

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

In the OIP-local provider this becomes an Observation with:

- `provenance.sourceType = "conversation_turn"`
- `provenance.speaker = "user"`
- `drefs.session = <sessionId>`
- `drefs.turnData = <sessionId>/<turnId>`

See [../memory/conversation-capture.md](../memory/conversation-capture.md).

## 4) There are two early exits before any LLM prompt is assembled

Right after the user turn is committed to memory, the orchestrator checks three
parsers:

```ts
const delegation = parseDelegateIntent(text);
if (delegation) { … return; }
if (looksLikeDocsIngestTask(text)) { … return; }
if (looksLikeXIngestTask(text)) { … return; }
```

If any of these fire, the system routes to a harness and returns without
building the normal prompt or generating a normal assistant reply.

So the canonical "talk → prompt → answer" loop is not universal. Some requests
are understood as commands to run a side pipeline instead.

See [../capabilities/delegation.md](../capabilities/delegation.md).

## 5) Prompt assembly includes persona, memory, capabilities, and reminders

The initial prompt assembly at `session.ts:395` is:

```ts
const prompt = this.promptAssembler.assemble({
  systemInstructions: this.config.systemInstructions,
  extraSystem: this.turnExtraSystem,
  currentUserTurn: text,
  recentConversation: this.recentTurns.slice(0, -1).map(...),
  personaContext: this.opts.personaContext,
  retrievedMemory: memory.items,
  availableCapabilities: conversationalCapabilities(this.opts.conversational),
  dueReminders: this.turnDueReminders,
  mode: "initial",
  lateAddenda: [],
  agentResults: [],
});
```

Important details:

- `recentConversation` excludes the current user turn because it was just pushed
  to `recentTurns`; hence `slice(0, -1)`
- `personaContext` is injected on every assembly call, not just the first
- due reminders are loaded just before assembly
- the same assembler is reused for later addendum and follow-up prompt builds

Full structure: [prompt-assembly.md](prompt-assembly.md).

## 6) Generation is tracked separately from a single string

Before calling the model, the orchestrator creates a response ledger bucket and
a tool-call counter:

```ts
const responseId = this.responseLedger.beginResponse(this.sessionId, turn.id);
const track = { toolcallCount: 0 };
const assistantText = await this.generateWithFailover(..., track);
await this.responseLedger.commit(responseId, assistantText);
```

The ledger is not just logging. It is the system's authoritative split between:

- proposed text
- committed text
- text sent to TTS
- buffered text
- delivered text
- abandoned text
- resumed text

Why that split exists: tool calls and interruptions can cause a response to be
built in multiple stages, with some text spoken and some not.

See [response-ledger.md](response-ledger.md).

## 7) Addenda after tool calls are merged into the committed text

After the first generation pass, the orchestrator checks:

```ts
if (this.pendingAddenda.length > 0) {
  await this.generateAddendumSegments(turn, responseId);
}
```

The merged-text behaviour matters because older behaviour could strand the
useful part of an answer in addendum segments after a tool call. On this branch,
addendum text is merged back into the response ledger's committed text before
final storage.

The relevant session.ts grep landmarks are:

- line 540: combine existing committed text with segment text
- line 541: `await this.responseLedger.commit(responseId, combined)`

This is the runtime side of the turn-lifecycle patch documented in the vault.

## 8) Delivery uses committed text, not raw first-pass text

After addenda have been merged, the orchestrator speaks the committed text:

```ts
const committed = this.responseLedger.getCommittedText(responseId);
const interrupted = await this.speak(responseId, turn.id, committed);
```

So the thing sent to delivery is the ledger's canonical post-merge string.

If delivery is interrupted, interruption handling takes ownership of the rest of
the lifecycle and the normal path returns early.

## 9) The persisted assistant turn uses delivered text if available

If delivery was not interrupted, the assistant turn object is built as:

```ts
const assistantTurn: ConversationTurn = {
  …
  text: this.responseLedger.getDeliveredText(responseId) || committed,
  …
};
```

That `|| committed` fallback means:

- if something was actually delivered, the persisted conversational turn matches
  what was spoken / shown
- otherwise it falls back to the full committed text

The distinction matters in interruption scenarios where only part of the reply
was delivered before cancellation.

## 10) The assistant turn is also committed into OIP memory, with metadata

After persistence, memory commit runs again:

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
    reasoningLineCount: assistantTurn.text.split("\n").length,
  },
});
```

In `packages/memory/src/oip-local/provider.ts:245+`, the assistant turn becomes
an Observation with:

- `provenance.sourceType = "conversation_turn"`
- `provenance.speaker = "assistant"`
- `drefs.session = <sessionId>`
- `drefs.turnData = <sessionId>/<assistantTurnId>`
- `alfred.toolcallCount = N`
- `alfred.reasoningLineCount = N`

This is how the OIP graph knows that a given assistant reply involved tool use
without embedding full tool-call transcripts into the observation prose.

## 11) Then the state machine moves back to a waiting state

If the orchestrator has not been cancelled:

- transition to `Listening` when allowed
- otherwise transition to `Idle` when allowed

This keeps turn completion separate from transport concerns — the session can be
alive and listening again without rebuilding the runtime.

## Design properties visible in this flow

### Retrieval-before-write for the current user turn
The system avoids retrieving the just-said text on the same turn.

### Both sides of the conversation are memory objects
This means Alfred's graph is not just a user-facts store or a docs store — it
is also a conversation archive.

### The response ledger is the source of truth for the assistant reply
Not the first generation string, and not just the delivered text.

### Tool use is summarized, not embedded verbatim, in assistant Observations
`toolcallCount` and `reasoningLineCount` are light metadata; the heavy detail
stays one lookup away via `turnData`.

## Related

- [prompt-assembly.md](prompt-assembly.md) — what gets assembled before
  generation
- [response-ledger.md](response-ledger.md) — proposed vs committed vs delivered
- [../memory/conversation-capture.md](../memory/conversation-capture.md) — how
  the turn commit maps into OIP records
- [../capabilities/delegation.md](../capabilities/delegation.md) — the early
  exits before generation
