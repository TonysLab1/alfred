# Response Ledger — Proposed, Committed, Delivered

The response ledger is Alfred's answer to a simple problem: an assistant reply
is not always one string created in one moment and spoken unchanged.

Tool calls, addenda, streaming delivery, interruption, abandonment, and resume
all split a "response" into stages. `packages/core/src/response-ledger.ts`
tracks those stages explicitly.

## Why it exists

The class comment says it directly:

> Tracks assistant response materialization separately from a single string.

Without this separation, the system would lose track of which text was merely
proposed, which was canonically committed, which was actually delivered, and
which had to be abandoned after interruption.

## The tracked buckets

`ResponseLedger` holds one set of buffers per response id:

- `segments`
- `proposed`
- `committed`
- `submitted`
- `buffered`
- `delivered`
- `abandoned`
- `resumed`
- `cancellations`
- `meta` (session id + turn id)

The corresponding snapshot shape,
`ResponseLedgerSnapshot`, exposes these under the names:

- `proposedText`
- `committedText`
- `submittedToTtsText`
- `bufferedText`
- `deliveredText`
- `unspokenText`
- `abandonedText`
- `resumedText`
- `cancellationReasons`

## Lifecycle of a response

### beginResponse

`beginResponse(sessionId, turnId, responseId?)` allocates empty buffers for a
new reply and returns the response id. This is called before generation starts.

### addSegment / appendProposed

Segments are appended as typed pieces of a response, and their text is also
added to the `proposed` bucket. Each append records a ledger row and emits a
`response.proposed` event.

### commit

`commit(responseId, text?)` chooses a canonical committed string for the
response. If no explicit text is passed, it uses whatever is in `proposed`.

This is the decisive step: the committed text is the thing later used for
speaking and, on normal completion, for memory commit.

### submitToTts / bufferAudio / markDelivered

These three steps track outward delivery:

- submitted to TTS
- audio buffered
- text delivered

Each appends to a different bucket and emits a different event.

### abandon / markResumed

If interruption or cancellation occurs, text can move into the `abandoned`
bucket with an explicit cancellation reason, and later a resumed path can be
marked.

## What the orchestrator uses it for

The main turn path in `session.ts` uses the ledger in three ways:

1. after generation, it commits the first-pass text
2. if addenda exist, it merges them into the committed text
3. when building the persisted assistant turn, it prefers delivered text and
   falls back to committed text

Key call sites from `session.ts`:

- line 386 — `beginResponse(...)`
- line 408 — initial `commit(responseId, assistantText)`
- lines 540–541 — merge addendum segment into the committed string and commit
  again
- line 415 — `getCommittedText(responseId)` before speaking
- line 426 — `getDeliveredText(responseId) || committed` when persisting the
  assistant turn

## Committed vs delivered is not a cosmetic distinction

These two strings can differ.

### When they are the same
Normal uninterrupted text turn: the full committed response is delivered.

### When they differ
Interruption after only part of a response has been spoken or streamed. In that
case:

- `committedText` is the full reply Alfred settled on
- `deliveredText` is the prefix that actually reached the user
- `getUnspokenRemainder()` returns the suffix that did not

`getUnspokenRemainder()` computes this by comparing `committed` and `delivered`.
If the delivered text is a prefix, it returns the suffix; if the two diverged it
falls back to a best-effort slice by delivered length.

This split is the difference between "what Alfred meant to say" and "what the
user actually heard."

## Why this matters for memory

The assistant turn later committed into OIP uses:

```ts
text: this.responseLedger.getDeliveredText(responseId) || committed
```

That means the stored conversational turn aims to reflect user-visible reality,
not just ideal internal completion. If only half the response was delivered,
that half is what gets persisted as the assistant turn text.

This is a subtle but important epistemic choice: the memory graph stores the
conversation as it happened, not merely as it would have happened without
interruption.

## Segments explain why addenda matter

A response can be made of multiple segments. Tool calls often produce this
shape:

1. first generation emits a lead-in ("let me check")
2. tool runs
3. a later generation emits the useful content

If only the first segment were committed to memory, the graph would contain the
opening phrase but not the actual answer. The current branch fixes this by
merging addenda into the committed string before delivery and storage.

See [turn-lifecycle.md](turn-lifecycle.md) for the surrounding sequence.

## Related

- [turn-lifecycle.md](turn-lifecycle.md) — where the ledger is used in the turn
- [../memory/conversation-capture.md](../memory/conversation-capture.md) — where
  committed assistant turns become OIP Observations
