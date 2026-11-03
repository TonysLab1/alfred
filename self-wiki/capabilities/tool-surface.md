# Tool Surface — All Callable Tools With Their Grades and Effects

Alfred's tools are defined in `apps/voice-agent/src/live/tools.ts` for the
GPT-Live voice path, and the desktop/text HTTP path provides an overlapping set
through `conversational-tools.ts` and port wiring. This document lists every
tool registered in the live path and notes which are also available in text
mode.

## Tool grades

Grades follow the A–E classification used in HomeBuddy's deterministic safety
policy (see also the table in the repository
`references/plan-alfred-self-wiki.md`). These are **documentation of stance,
not runtime enforcement.** Nothing in the code consults these grades before
execution.

| Grade | Meaning | Risk profile |
|---|---|---|
| A | Read-only, no side effects | Safe to call at any time |
| B | Writes local state | Mutates Alfred's own data, not the world |
| C | Affects the physical/external world | Lights, audio, briefings — observed by people |
| D | Delegates to an external harness | Runs user-supplied commands or triggers workflows |
| E | Blocked — should never run without explicit authorization | None currently registered |

## The live voice tool set

Tools registered in `createAlfredLiveTools()` at
`apps/voice-agent/src/live/tools.ts`:

| Tool | Grade | Effect | Text mode? |
|---|---|---|---|
| `search_memory` | A | Queries OIP memory: FTS + graph walk | Yes |
| `remember_memory` | B | Writes durable entities, assertions, notes into OIP | Yes |
| `update_reminder` | B | Marks a reminder as done, stopped, or snoozed | Yes |
| `list_due_reminders` | A | Lists reminders due now | Yes |
| `get_current_time` | A | Returns current time for a named place, or home | Yes |
| `get_weather_forecast` | A | Weather forecast with optional location | Yes |
| `get_news_headlines` | A | Current news headlines | Yes |
| `summarize_news_article` | A | Follow-up detail on a specific headline | Yes |
| `get_crypto_price` | A | Bitcoin and other crypto prices | Yes |
| `get_metals_price` | A | Gold and silver prices | Yes |
| `get_earthquakes` | A | Recent earthquake report | Yes |
| `get_weather_alerts` | A | Severe weather warnings / advisories | Yes |
| `get_space_weather` | A | Geomagnetic storms, aurora conditions | Yes |
| `get_natural_events` | A | Wildfires, volcanoes, other natural events | Yes |
| `get_exchange_rate` | A | Currency conversion rates | Yes |
| `get_hacker_news` | A | Hacker News front page | Yes |
| `control_studio_lights` | C | Turn Elgato Key Lights on/off, adjust brightness/temperature | Yes |
| `play_daily_briefing` | C | Play the briefing greeting with weather, news, reminders | Voice only |
| `decline_briefing_offer` | B | Decline the soft-offer briefing prompt | Yes |
| `show_expression` | C | Drive the AlfredBot robot face: smile, frown, wink, etc. | Voice only |
| `delegate_task` | D | Dispatch a task to an external harness by category | Yes |

Additional tools available only when the archive port is connected (see
[../identity/persona-override.md](../identity/persona-override.md)):

| Tool | Grade | Effect |
|---|---|---|
| `search_archive` | A | Full-text or structured search across archive documents |
| `open_archive_record` | A | Open a specific archive record by id |
| `archive_neighbors` | A | Records near a given record in the archive |
| `ask_archive` | A | Natural-language query against the archive |

## How the LLM learns to use tools

The `PromptAssembler` generates capability guidance per tool. Each tool entry in
the guidance block tells the LLM:

- when to call the tool ("casual phrasing like 'what's the weather'")
- what data it provides
- what the tool prefers over `delegate_task`
- what it must not invent or guess

Example from the assembler (see
[../runtime/prompt-assembly.md](../runtime/prompt-assembly.md)):

```
When the user asks about current weather or the forecast, call
get_weather_forecast immediately (including casual phrasing like
"what's the weather"). Do not ask which city or zip unless they
named a different place — if they omit a location, call the tool
with no location argument so home is used. Do not invent
temperatures or conditions.
```

In the live voice path, `buildLiveVoiceInstructions` defines a different set of
rules for the voice model and `buildLiveBackendInstructions` defines the tool
list for the backend handler.

## What is NOT a tool

- `read_file` / `search_files` — not registered. Alfred cannot read arbitrary
  files on disk during a conversation turn. See
  [../identity/persona-files.md](../identity/persona-files.md) for the file
  loading that happens at startup.
- `terminal` / `execute_code` — not registered. Alfred cannot run shell commands
  or execute Python directly.
- `web_search` — not registered. Alfred cannot search the web during a
  conversation turn (the news/markets tools provide structured data from known
  APIs, not open web queries).

## Privacy note on tool data

Weather and time tools use the configured home location. Reminder data is stored
in OIP. Searchable memory includes past conversations. `remember_memory` stores
structured assertions permanently. None of these tools phone home —
`alfrd.net` provides cloud sync (see [../surfaces/README.md](../surfaces/README.md))
but only when enabled.

## Related

- [delegation.md](delegation.md) — the `delegate_task` routing system
- [../runtime/prompt-assembly.md](../runtime/prompt-assembly.md) — the guidance
  blocks
- [../memory/retrieval.md](../memory/retrieval.md) — the `search_memory` backend