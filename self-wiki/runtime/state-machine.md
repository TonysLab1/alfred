# State Machine — The Legal Transition Table

Every session's life is governed by a single explicit transition table in
`packages/core/src/state-machine.ts`. The table is intentional and inspectable —
no dynamic construction, no implicit edges, no event-driven loose coupling.

## The complete transition table

From the `ALLOWED` constant:

```
Idle                          → Listening, Cancelled
Listening                     → UserSpeechDetected, Cancelled, Failed
UserSpeechDetected            → Transcribing, UserTurnCommitted, Cancelled, Failed
Transcribing                  → UserTurnCommitted, Failed, Recovering, Cancelled
UserTurnCommitted             → RetrievingMemory, GeneratingResponse, AgentTaskDelegated, Cancelled
RetrievingMemory              → GeneratingResponse, AgentTaskDelegated, Failed, Recovering, Cancelled
GeneratingResponse            → UserAddendumReceived, SynthesizingSpeech, AssistantSpeaking,
                                 AgentTaskDelegated, Failed, Recovering, Cancelled
UserAddendumReceived          → GeneratingResponse, Cancelled, Failed
SynthesizingSpeech            → AssistantSpeaking, Failed, Recovering, Cancelled
AssistantSpeaking             → UserBackchannelReceived, GenuineInterruptionReceived,
                                 GeneratingResponse, Idle, Listening, Cancelled, Failed
UserBackchannelReceived       → AssistantSpeaking, Listening, Idle, Cancelled
GenuineInterruptionReceived   → InterruptionArbitration, Cancelled, Failed
InterruptionArbitration       → GeneratingResponse, ResponseResumption, AssistantSpeaking,
                                 Listening, Idle, Cancelled, Failed
ResponseResumption            → SynthesizingSpeech, AssistantSpeaking, GeneratingResponse, Cancelled
AgentTaskDelegated            → WaitingForAgentResult, Cancelled, Failed
WaitingForAgentResult         → GeneratingResponse, Failed, Recovering, Cancelled
Failed                        → Recovering, Idle, Listening, Cancelled
Recovering                    → Idle, Listening, GeneratingResponse, Cancelled, Failed
Cancelled                     → Idle
```

19 states, with the total vocabulary defined as `ConversationState` in
`@alfred/contracts`.

## Design properties visible in the table

### Recovery and cancellation are broad edges
Every state except `Cancelled` itself has `→ Cancelled` as a legal transition.
That is the escape hatch — no matter where the session is, it can be cancelled
and cleanly moved to `Idle`.

`Failed` and `Recovering` are similarly permissive in their outgoing edges. A
session in `Failed` can go to `Recovering`, `Idle`, `Listening`, or `Cancelled`
— giving the orchestrator room to retry, reset, or shut down.

### Transcription can be skipped
`UserSpeechDetected → UserTurnCommitted` is legal. Text-mode sessions never
enter `Transcribing` — text input passes through directly.

### Agent delegation can occur from three states
`UserTurnCommitted`, `RetrievingMemory`, and `GeneratingResponse` all have
`→ AgentTaskDelegated` edges. This means delegation can skip retrieval, skip
generation, or interrupt generation once it has started.

### Multiple paths to finish a turn
`AssistantSpeaking` can transition to `Idle` (turn done, no follow-up) or
`Listening` (turn done, wait for next). The choice depends on the surface:
voice surfaces typically go to `Listening`, text surfaces to `Idle`.

## Transition method

`transition(to, trigger, opts?)` checks `canTransition(to)` and throws
`InvalidTransitionError` on illegal edges. On success it emits a
`state.transition` event with `{from, to, trigger}` in the payload, plus
optional `turnId`, `responseId`, `correlationId`, `causationId` for tracing.

`force(to, trigger, opts?)` bypasses the legality check and writes the state
with `forced: true` in the event payload. This is for recovery paths that must
leave `Cancelled`/`Failed` cleanly, where the guard conditions don't apply.

## Where this matters for self-knowledge

Understanding the state machine is the difference between "why didn't Alfred
respond" and reading the code. If someone reports Alfred stuck in `Idle` despite
a user utterance, the question is "which transition failed?" — and this table is
the answer key.

## Related

- [turn-lifecycle.md](turn-lifecycle.md) — the orchestrator that drives these
  transitions
- [failover.md](failover.md) — the provider failover controller that can produce
  `Failed` states