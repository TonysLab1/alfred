# Simulator — Integration Test Suite, Not Just a CLI

The simulator at `apps/simulator/` is a full integration test harness that runs
the real `SessionOrchestrator` against a complete fake I/O stack. It is not a
lightweight chat loop — it is the primary automated test suite for the
conversation core.

Entry point: `apps/simulator/src/cli.ts`.
World construction: `apps/simulator/src/harness.ts`.
Scenarios: `apps/simulator/src/scenarios.ts`.

## Architecture

`createWorld()` builds a complete, self-contained Alfred session with no
external dependencies:

- `FakeClock` — deterministic time, no real delays
- `FakeLLMProvider` — programmatic replies, configurable failures
- `FakeSTTProvider` / `FakeTTSProvider` — text passthrough
- `FakeUnifiedProvider` — for unified pipeline mode tests
- `FakeMemoryProvider` — seeded memory for retrieval tests, plus
  `LocalFileMemoryProvider` registered as a secondary
- `FakePersistence` (`createInMemoryPersistence`) — no disk writes
- Four stub harnesses (openclaw, hermes, codex, claude) with configurable
  failure counts
- `SessionOrchestrator` from `packages/core` — the real thing, just with fake
  dependencies

Profile: `"profile.demo"`. Active memory: `"memory.fake"`.

## The 15 scenarios

Each scenario constructs a world, runs turns through the orchestrator, and
asserts on events and state. Categories covered:

| # | Name | Tests |
|---|---|---|
| 01 | Normal conversational turn | User utterance → `turn.committed` event → Listening/Idle state |
| 02 | Memory retrieval in prompt | Seeded memory → `memory.retrieved` event → items in retrieved payload |
| 03 | LLM primary failure + sticky fallback | Primary fails ×2 → switches to secondary → stays sticky |
| 04 | Manual primary check | Health probe → restore primary → verify active provider |
| 05 | Timeout-based primary restore | Advance fake clock past retry interval → primary auto-restored |
| 06 | Addendum during generation | `asAddendum` utterance mid-generation → addendum segment in response |
| 07 | Genuine interruption | `forcedArbitration: abandon_and_answer` → interruption + abandonment events |
| 08 | Backchannel passes through | `utteranceKind: backchannel` during speech → backchannel event, no interruption |
| 09 | Interruption abandons remainder | Abandoned response → `response.abandoned` event |
| 10 | Finish sentence then answer | `forcedArbitration: finish_sentence_then_answer` → correct outcome |
| 11 | Delegation by category | `delegate:coding:…` format → agent harness executes |
| 12 | Agent harness failover | OpenClaw stub fails → Hermes stub takes over |
| 13 | Unified pipeline locks selectors | Unified mode → all selectors locked with `unified_mode_active` |
| 14 | Switch unified→cascaded | Pipeline update → selectors unlocked, mode is cascaded |
| 15 | Active memory provider switching | Switch from fake to fake.alt → active provider changes |

Scenarios 03–05 exercise the provider failover / stickiness / auto-recovery
circuit. Scenarios 06–10 test the interruption arbitration model (different
arbitration strategies, backchannel vs. genuine). Scenarios 11–12 test
delegation and agent routing. Scenarios 13–14 test the unified pipeline mode.

## Persona and memory

The simulator does NOT load persona files from disk. `createWorld()` passes
`personaContext: undefined` (not set), so the session operates without
SOUL/IDENTITY/USER. The system instructions are hardcoded:

```ts
systemInstructions: "You are ALFRED. Be concise. Prefer delegate_task for external actions."
```

Memory is purely seeded in the fake provider. The OIP provider is NOT registered
— only `FakeMemoryProvider`, `FakeMemoryProvider` (second instance), and
`LocalFileMemoryProvider` (at `/tmp/alfred-m1-memory.jsonl`).

This means the simulator cannot verify wiki ingestion or OIP retrieval. For
those, use the desktop or voice surfaces.

## Running

```bash
pnpm simulate
# or: pnpm --filter @alfred/simulator start
# or: tsx apps/simulator/src/cli.ts
```

Output: `[PASS|FAIL] <scenario-id> <name> — <detail>`, then a summary.

## `awaitWithClock`

The harness includes a helper that prevents deadlock when using a fake clock:
it calls `world.clock.advance(step)` in a loop while waiting for session
promises to settle. Without this, work that registers timers would never resolve
in a fake-clock environment.

## Why this matters for self-knowledge testing

The simulator proves the session orchestrator's correctness for 15 distinct
operational paths, all using deterministic fake time. But it does NOT test:

- persona loading or the HYPATIA override
- OIP memory commit or retrieval
- document ingestion
- real voice pipeline behaviour
- any surface-specific routing

Persona and memory tests live in their own test suites
(`packages/memory/src/persona.test.ts`, `oip-local.test.ts`, etc.). The
simulator tests the orchestrator.

## Related

- [../runtime/turn-lifecycle.md](../runtime/turn-lifecycle.md) — the
  orchestrator being tested
- [../identity/persona-loading.md](../identity/persona-loading.md) — the
  persona path the simulator does NOT exercise
- [../memory/retrieval.md](../memory/retrieval.md) — the retrieval the
  simulator mocks