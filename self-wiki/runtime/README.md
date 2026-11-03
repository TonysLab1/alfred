# Runtime — How a Turn Actually Runs

What happens between someone speaking and Alfred answering. This area covers
the session orchestrator, the prompt assembler, the response ledger, and the
state machine that governs transitions.

## Documents in this area

| Document | What it covers |
|---|---|
| [turn-lifecycle.md](turn-lifecycle.md) | The full sequence: transcribe, commit, retrieve, assemble, generate, speak, commit again |
| [state-machine.md](state-machine.md) | The complete legal-transition table across all 19 states |
| [prompt-assembly.md](prompt-assembly.md) | How the system prompt is built, in what order, and what is injected unconditionally |
| [intent-layer.md](intent-layer.md) | How tool calls, delegation, and generation are dispatched before and during a turn |
| [failover.md](failover.md) | Provider stickiness, cooldown, health probing, and auto-restore |
| [response-ledger.md](response-ledger.md) | Segments, addenda, committed vs delivered text, and why the distinction matters |

## The shape of it

The orchestrator is `SessionOrchestrator` in `packages/core/src/session.ts`.
One method drives a turn; the sequence is roughly:

```
transcribe → commit user turn → retrieve memory → commit user turn to memory
          → [possible delegation short-circuit]
          → assemble prompt → generate (with failover + tool calls)
          → process addenda → speak → commit assistant turn to memory
```

Each step emits events and most are gated by a state machine transition.

Two things worth knowing up front:

- **Both the user turn and the assistant turn are committed to memory.** Each
  becomes an OIP Observation. See
  [../memory/conversation-capture.md](../memory/conversation-capture.md).
- **Delegation can short-circuit the whole generation path.** If the user text
  parses as a delegation intent or looks like an ingest task, the turn routes
  to a harness and returns before any prompt is assembled. See
  [../capabilities/delegation.md](../capabilities/delegation.md).

## Where persona enters

The orchestrator holds `personaContext` as a constructor option and passes it
into every `promptAssembler.assemble()` call — four call sites in `session.ts`
(lines 397, 517, 703, 797) corresponding to the initial generation and the
various addendum and post-delegation paths.

Persona is therefore loaded once at construction and re-injected on every
assembly. See [../identity/persona-loading.md](../identity/persona-loading.md)
for the caching consequence.

## Related

- [../memory/README.md](../memory/README.md) — the retrieve and commit steps
- [../capabilities/README.md](../capabilities/README.md) — tools and delegation
  invoked during generation
- [../surfaces/README.md](../surfaces/README.md) — the different front ends that
  drive this loop
