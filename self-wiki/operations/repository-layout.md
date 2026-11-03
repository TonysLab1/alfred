# Repository Layout — Where Everything Lives

Alfred is a pnpm monorepo. Root: `/home/controlroomtoon/alfred/`.

## Top-level

```
alfred/
├── packages/         ← The shared core: memory, contracts, agents, persistence
├── apps/             ← Runtime surfaces: voice-agent, desktop-client, simulator, etc.
├── data/             ← User data: persona, memory providers, conversations, briefing
├── self-wiki/        ← This self-knowledge documentation
├── docs/             ← Public-facing documentation
├── node_modules/     ← Dependencies
├── package.json      ← Root workspace; scripts for all apps
├── pnpm-workspace.yaml
├── tsconfig.base.json
├── vitest.config.ts
├── .env              ← Environment variables
├── .gitignore        ← Excludes data/, node_modules, dist
├── Makefile
├── docker-compose.yml
└── scripts/          ← Helper scripts
```

## packages/

Each package is self-contained with its own `package.json`, `src/`, and tests.

| Package | Purpose |
|---|---|
| `packages/core/` | `SessionOrchestrator`, `PromptAssembler`, `ResponseLedger`, state machine |
| `packages/memory/` | `MemoryController`, `OipLocalMemoryProvider`, OIP internals, docs-ingest, persona loader |
| `packages/agents/` | `AgentRouter`, harness implementations and stubs |
| `packages/contracts/` | Zod schemas and types shared across packages |
| `packages/persistence/` | Storage backends for turns, sessions, settings |
| `packages/briefing/` | Briefing controller for daily greeting |
| `packages/livekit/` | LiveKit media bridge integration |
| `packages/elgato/` | Elgato Key Light control |
| `packages/conversations/` | Conversation index and history |
| `packages/browser/` | Playwright-based browser capture for X ingest |
| `packages/calendar-gateway/` | Apple/iOS calendar access |
| `packages/provider-*` | LLM provider implementations (OpenAI, Deepgram, ElevenLabs, xAI, Apple STT) |

## apps/

| App | Purpose |
|---|---|
| `apps/voice-agent/` | Primary voice interface: cascade and GPT-Live paths |
| `apps/desktop-client/` | HTTP text API and web UI |
| `apps/simulator/` | CLI turn simulator for testing |
| `apps/voice-client/` | Browser-based voice frontend |
| `apps/alfredbot/` | Discord/social bot connector |
| `apps/iOS-client/` | Mobile app |

## data/

`data/` is **gitignored** (`.gitignore` line 15: `**/data/`). Everything here
is local to the installation.

```
data/
├── persona/<profileId>/    ← SOUL.md, IDENTITY.md, USER.md
├── memory-oip/<profileId>/ ← OIP package store + SQLite indexes
├── memory/<profileId>/     ← JSONL provider + ingest ledger/sources
├── conversations/           ← Conversation index
└── briefing/<profileId>/   ← Briefing state and cache
```

Current profile: `profile.default` (set by `ALFRED_PROFILE_ID` in `.env`).

## Git tracking boundary

- **Tracked:** everything in `packages/`, `apps/`, `self-wiki/`, config at root
- **Not tracked:** `data/`, `node_modules/`, `dist/`, `.env`
- **Self-wiki:** tracked at `self-wiki/`, not under `data/` — deliberately
  placed outside the gitignore to make it reviewable and version-controlled

## Related

- [../memory/oip/record-model.md](../memory/oip/record-model.md) — OIP on-disk
  layout
- [../memory/docs-ingestion.md](../memory/docs-ingestion.md) — where the ledger
  lives
- [../identity/persona-files.md](../identity/persona-files.md) — persona file
  location