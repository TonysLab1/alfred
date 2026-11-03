# Delegation — How Alfred Dispatches Work to Harnesses

When the conversational model cannot handle a task directly — because it needs
coding tools, a browser, email, or calendar access — it calls `delegate_task`.
That call goes through `AgentRouter` at `packages/agents/src/router.ts`, which
finds a harness for the task category and attempts execution with automatic
failover.

## The router

`AgentRouter` is a thin mapping from task categories to ordered harness lists.

```ts
class AgentRouter {
  private harnesses = new Map<string, AgentHarness>();
  private rules: AgentRoutingRule[];
}
```

Where `AgentRoutingRule` is:

```ts
interface AgentRoutingRule {
  category: TaskCategory;
  orderedHarnessIds: string[];
}
```

### delegate()

When `delegate(request) `is called:

1. the router looks up the ordered harness ids for the requested task
   category from its rules
2. for each harness id, it finds the harness, checks it `supports()` the
   category, and calls `execute(request)`
3. the first harness that returns a non-`failed` result wins — further harnesses
   in the chain are not tried
4. if every harness in the chain fails, the last failure is returned or a
   default "All harnesses failed" error

## Categories and their registered harnesses

The task category vocabulary is a loose string union:

- `household` — Alfred Home harness
- `calendar` — Calendar gateway harness or OpenClaw local
- `coding` — OpenClaw or Codex or Claude
- `email` — OpenClaw or Hermes
- `research` — Docs ingest or X ingest or OpenClaw
- `browser` — X ingest or Hermes or Claude
- `computer_use` — Alfred Home or X ingest or Claude
- `general` — Alfred Home or OpenClaw

### Voice agent (brain.ts) routing rules

```ts
agentRouting: [
  { category: "coding", orderedHarnessIds: ["harness.codex"] },
  { category: "email", orderedHarnessIds: ["harness.openclaw", "harness.hermes"] },
  { category: "research", orderedHarnessIds: ["harness.docs-ingest", "harness.x-ingest", "harness.hermes"] },
  { category: "browser", orderedHarnessIds: ["harness.x-ingest", "harness.hermes"] },
  { category: "computer_use", orderedHarnessIds: ["harness.x-ingest", "harness.claude"] },
  { category: "calendar", orderedHarnessIds: ["harness.openclaw-local"] },
]
```

### Desktop text session routing rules

More categories, and different chains:

```ts
agentRouting: [
  { category: "household", orderedHarnessIds: ["harness.alfred-home"] },
  { category: "calendar", orderedHarnessIds: ["harness.openclaw-local"] },
  { category: "coding", orderedHarnessIds: ["harness.openclaw"] },
  { category: "email", orderedHarnessIds: ["harness.openclaw"] },
  { category: "computer_use", orderedHarnessIds: ["harness.alfred-home", "harness.x-ingest"] },
  { category: "research", orderedHarnessIds: ["harness.docs-ingest", "harness.x-ingest", "harness.openclaw"] },
  { category: "browser", orderedHarnessIds: ["harness.x-ingest", "harness.openclaw"] },
  { category: "general", orderedHarnessIds: ["harness.alfred-home", "harness.openclaw"] },
]
```

Note the desktop path has `household` and `general` categories that the voice
path does not, and uses `harness.openclaw` for `coding` instead of `codex`.

### The Hermes stub

`harness.hermes` appears as a fallback position in `email`, `research`, and
`browser` for the voice agent. This harness is a stub — it is registered but its
behaviour depends on the local configuration. The point is that it exists in the
chain as the last-resort attempt before failure.

## Stub harnesses

`packages/agents/src/stub-harness.ts` provides factory functions:

```ts
createHermesStub() → AgentHarness
createCodexStub()   → AgentHarness
createClaudeStub()  → AgentHarness
createOpenClawStub() → AgentHarness
```

A stub harness has a manifest with a `fallbackHint` and returns a default
"unavailable" result. It exists so that the router can include the harness in
a failover chain without requiring it to actually be running.

## Real harnesses

The implemented harnesses (not stubs):

- **OpenClaw** (`openclaw-harness.ts`) — Local process execution via OpenClaw
  CLI. Supports coding, email, calendar (local), general.
- **Alfred Home** (`home-harness.ts`) — HTTP callback to alfred-home service
  for household actions, lights, calendar, general.
- **Docs Ingest** (`docs-ingest-harness.ts`) — Document ingestion pipeline
  trigger. Supports research category.
- **X Ingest** (`x-ingest-harness.ts`) — X/Twitter capture via Playwright.
  Supports browser, research, computer_use.
- **Calendar Gateway** (`calendar-gateway-harness.ts`) — Apple/iOS calendar
  via the alfred-calendar gateway package. Supports calendar.

## Early delegation exits in the turn lifecycle

Before prompt assembly, the orchestrator runs three checks:

```ts
const delegation = parseDelegateIntent(text);
if (delegation) { … return; }
if (looksLikeDocsIngestTask(text)) { … return; }
if (looksLikeXIngestTask(text)) { … return; }
```

`parseDelegateIntent` recognizes text formatted as `delegate:category:description`
— a lightweight text intent rather than a tool call. `looksLikeDocsIngestTask`
and `looksLikeXIngestTask` match the user's natural-language phrasing against
patterns that indicate the task should be routed directly to the docs-ingest or
x-ingest pipeline.

These are short-circuit exits: if any match, the orchestrator calls
`runDelegation()` and returns without building the normal prompt or generating
a conversational assistant turn.

## Properties of the delegation system

### The LLM does not decide which harness
The LLM calls `delegate_task` with a category and text. The router resolves the
category to a harness. The LLM never sees harness-specific tool schemas.

### Failover is automatic within a turn
If the first harness fails, the router tries the next one in the same turn.
The LLM's original text is reused; it does not need to re-delegate.

### There is no policy gate
The router executes immediately when called. Nothing evaluates the task
category, the requested action, or the user's authority before dispatching.

### Stubs prevent hard failures
If the configured primary harness is unavailable, a stub can return a graceful
"unavailable" result instead of throwing, keeping the conversation alive.

## Related

- [tool-surface.md](tool-surface.md) — the full tool list, including
  `delegate_task`
- [../runtime/turn-lifecycle.md](../runtime/turn-lifecycle.md) — where delegation
  short-circuits
- [../runtime/prompt-assembly.md](../runtime/prompt-assembly.md) — the
  `delegate_task` guidance block