# Operations — Repo Layout, CLI, and Verification

Practical reference: where files live on disk, how to run maintenance commands,
and how to verify that a change actually took effect.

## Documents in this area

| Document | What it covers |
|---|---|
| [repository-layout.md](repository-layout.md) | The monorepo map: packages, apps, data directories |
| [memory-cli.md](memory-cli.md) | The `pnpm memory` command surface: ingest-docs, oip-inspect, rebuild, cleanup |
| [change-and-verification.md](change-and-verification.md) | The procedure for making a change and proving it worked: reset, ingest, test |
| [environment.md](environment.md) | Environment variables that control Alfred's behaviour |

## Why this area is thin

The repo layout matters because self-knowledge needs to know where things live.
The CLI matters because docs ingestion and persona change verification both
require it. The change procedure matters because persona is cached per-runtime
and docs ingestion is incremental — both can produce false negatives if
verified without the right steps.

## Related

- [../memory/docs-ingestion.md](../memory/docs-ingestion.md) — the pipeline
  invoked by `pnpm memory -- ingest-docs`
- [../identity/persona-loading.md](../identity/persona-loading.md) — the caching
  that makes verification tricky
- [../runtime/turn-lifecycle.md](../runtime/turn-lifecycle.md) — the session
  model