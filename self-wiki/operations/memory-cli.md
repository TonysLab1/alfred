# Memory CLI — Maintenance Commands

The `pnpm memory` command surface is defined in
`apps/voice-agent/src/memory-cli.ts`. All commands run against a profile,
defaulting to `ALFRED_PROFILE_ID` from `.env`.

## Ingest commands

### Register a docs source

```bash
pnpm memory -- ingest-docs-source add --path "/abs/path" [--label "label"]
```

Writes to `data/memory/<profileId>/ingest/docs-sources.json`.
Supports `.md`, `.txt`, `.rtf`, `.pdf` files.

### List registered sources

```bash
pnpm memory -- ingest-docs-source list
```

### Remove a source

```bash
pnpm memory -- ingest-docs-source remove --path "/abs/path"
```

### Run docs ingestion

```bash
pnpm memory -- ingest-docs [--path "/abs/path"] [--dry-run]
```

If `--path` is omitted, runs all registered sources. With `--dry-run`, reports
what would be ingested without writing to OIP.

Output per file: `[ingested|skipped|failed] path (N sections, M extracted)`.

## OIP maintenance

### Inspect a record

```bash
pnpm memory -- oip-inspect <record-id>
```

### Verify store integrity

```bash
pnpm memory -- oip-verify
```

Checks manifest-revision consistency, content hashes, detects orphans.

### Rebuild indexes from package store

```bash
pnpm memory -- oip-rebuild
```

Deletes and recreates the SQLite index from all current package revisions.
Use when the index is out of sync with the on-disk packages.

### Export OIP to JSON

```bash
pnpm memory -- oip-export [--output /path]
```

### Import OIP from JSON

```bash
pnpm memory -- oip-import <file>
```

### Bundle export and merge

```bash
pnpm memory -- oip-bundle-export
pnpm memory -- oip-bundle-merge <file.zip>
```

For cross-instance memory transfer.

## Persona maintenance

### List persona state

```bash
pnpm memory -- persona
```

### Export persona to file

```bash
pnpm memory -- export
```

### Import persona from file

```bash
pnpm memory -- import <file>
```

## Knowledge ingest

```bash
pnpm memory -- ingest-knowledge <file>
pnpm memory -- ingest-export <file>
```

## X/Twitter ingest

```bash
pnpm memory -- ingest-x-source add|list|remove
pnpm memory -- ingest-x-login
pnpm memory -- ingest-x
```

## User model maintenance

### Deduplicate USER.md

```bash
pnpm memory -- dedupe-user
```

Runs `dedupeUserMd()` to collapse near-duplicate bullets and paragraphs.

### Rebuild USER.md to budget

```bash
pnpm memory -- cleanup-user
```

Runs `cleanupUserMd()` — strips junk, collapses ingest blocks, demotes overflow
to memory notes. Soft target: 75% of 12,000 chars; hard ceiling: 92%.

### Erase

```bash
pnpm memory -- erase
```

## Seeding

```bash
pnpm memory -- seed-reminder [text]
```

Seeds a due reminder into OIP for the current briefing day.

## Full command list

```
Usage: pnpm memory -- inspect|persona|export|import|ingest-export|
       ingest-knowledge|ingest-x-source|ingest-x-login|ingest-x|
       ingest-docs-source|ingest-docs|dedupe-user|cleanup-user|erase|
       oip-inspect|oip-verify|oip-rebuild|oip-export|oip-import|
       oip-bundle-export|oip-bundle-merge|seed-reminder
```

## Related

- [../memory/docs-ingestion.md](../memory/docs-ingestion.md) — the pipeline
  behind `ingest-docs`
- [../identity/user-model.md](../identity/user-model.md) — the machinery behind
  `cleanup-user`
- [../memory/oip/provider.md](../memory/oip/provider.md) — the provider behind
  `oip-*` commands