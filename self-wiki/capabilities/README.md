# Capabilities — Tools, Harnesses, and Delegation

Alfred's action surface is shaped by two things: a set of domain-specific tools
that run locally, and a delegation router that dispatches sub-tasks to external
harnesses. This area covers both.

## Documents in this area

| Document | What it covers |
|---|---|
| [tool-surface.md](tool-surface.md) | The full list of callable tools with their grades and effects |
| [delegation.md](delegation.md) | How `delegate_task` works: the router, harnesses, failover chains, and categories |

## The shape of Alfred's action system

Alfred's tools are not an LLM tool-use free-for-all. They are purpose-built
functions registered on the backend response model, each with a specific tool
name and input contract. The GPT-Live voice path adds another set, mostly
overlapping.

Delegation is the way tasks that need external coordination — coding, email,
browser actions, calendar access — reach a harness. `delegate_task` routes
through `AgentRouter`, which tries harnesses in a configured order with
automatic failover.

## Key design properties

### Tools run on Alfred's machine, harnesses run elsewhere

`search_memory`, `control_studio_lights`, `play_daily_briefing` execute in
Alfred's own process. `delegate_task` sends work to a harness that may be
running in a different process or on a different machine.

### The router does not expose harness tool schemas to the LLM

The conversational LLM sees `delegate_task`, not the individual harness tools.
The router decides what harness to call based on category, not the LLM. This
is a deliberate separation: the conversational model delegates intent; the
router resolves it.

### Failover is per-category, per-harness priority

Each category has an ordered list of harnesses. If the first fails, the router
tries the second in the same turn. This is how the Hermes stub exists alongside
the active harness in some categories — a fallback, not a co-executor.

### There is no policy gate between the LLM and execution

When the conversational model calls `delegate_task`, the router runs it.
Nothing evaluates the safety, prominence, or urgency of the request before
execution. Tool grades in [tool-surface.md](tool-surface.md) are documentation
of a stance, not runtime enforcement.

## Related

- [../runtime/turn-lifecycle.md](../runtime/turn-lifecycle.md) — where delegation
  short-circuits the normal generation path
- [../runtime/prompt-assembly.md](../runtime/prompt-assembly.md) — how the
  capability guidance block teaches the LLM to use these tools
- [../surfaces/README.md](../surfaces/README.md) — which tools are available on
  which surface