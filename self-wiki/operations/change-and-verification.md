# Change and Verification — How to Make a Change and Prove It Worked

The procedure for changing Alfred and confirming the change took effect. This
is the document you consult when a developer says "it should be working now"
and you need to check.

## The two sources of false negatives

### Persona is cached per-runtime

Editing SOUL.md, IDENTITY.md, or USER.md on disk changes nothing in a running
Alfred session. Persona text is loaded once at runtime construction and cached.

**Fix:** call `resetTextSession()` or restart the process. For the desktop path,
`resetTextSession` deletes the runtime from the module-level `Map` so the next
`getTextSession` triggers a full rebuild including persona reload. See
[../identity/persona-loading.md](../identity/persona-loading.md).

### Docs ingestion is incremental

Running `pnpm memory -- ingest-docs` a second time with the same files reports
`[skipped]` for every file. This is correct behaviour — the ledger has
recorded the content hash and determined nothing changed. But it looks like
silence if you were expecting to see ingestion output.

**The tell:** the ledger at `data/memory/profile.default/ingest/docs-ledger.jsonl`
should be non-empty. An empty ledger after a run means ingestion touched zero
files — either because the path was wrong, the `data/` SKIP_DIRS gotcha fired,
or no markdown files were found.

## Verification procedure for self-wiki ingestion

### 1. Verify ingestion actually happened

```bash
wc -l data/memory/profile.default/ingest/docs-ledger.jsonl
```

Zero lines → ingestion did not find any files or was never run.
Non-zero → check the `[ingested]` / `[skipped]` counts in the run output.

### 2. Verify the Collection entity exists

```bash
pnpm memory -- oip-inspect <collection-did>
```

Or search for it from a live session: `search_memory("self-knowledge")` should
return records with `source=docs` in the metadata suffix.

### 3. Verify a specific wiki page is retrievable

From a running session, ask a question that should match a wiki page.
For example: "how does the turn lifecycle work" should trigger the FTS match
and the `docs_folder` booster. Check that the response references wiki content.

If retrieval returns conversation memory but no wiki content:

- the question may not match the FTS5 tokens in the wiki text (see
  [../memory/retrieval.md](../memory/retrieval.md) for the exact `toFtsQuery`
  tokenisation)
- the routing instruction in SOUL.md may not be triggering a memory search
  for self-referential questions
- the limit of 5 retrieved items may be crowded out by other memory

### 4. Verify a persona change took effect

After editing a persona file:

```bash
# Option A: reset the running session
# (HTTP endpoint, or programmatic call)
POST /api/conversation/reset

# Option B: restart the process
# (stop and restart pnpm voice / pnpm desktop)
```

Then send a turn that should reflect the changed persona text.

**Failure mode to watch for:** the old persona response and the new persona
response are indistinguishable. This does not mean the edit had no effect —
it may mean the edit touched only text that is never directly elicited by a
conversational turn. Test with a prompt designed to surface the change.

## Verification procedure for a code change

### 1. Check the branch

```bash
git branch --show-current
# expect: feat/memory-graph-nodes
```

### 2. Check working tree cleanliness

```bash
git status --short
# expect: nothing (or only intended changes)
```

### 3. Run the relevant test suite

```bash
pnpm test
# or a specific test file:
pnpm test packages/memory/src/oip-local/oip-local.test.ts
```

### 4. Verify the change in a live turn

Start Alfred in the relevant surface and send a turn that exercises the changed
code path. A test passing does not mean the change is observable in a real
session.

## The persona-reset gotcha in detail

`getTextSession` in `apps/desktop-client/src/lib/text-session.ts`:

```ts
let runtime = runtimes.get(key);
if (!runtime) {
  runtime = await buildRuntime(key, parts);
  runtimes.set(key, runtime);
}
```

`buildRuntime` calls `loadPersonaContext(dir)` which reads the three files.
Once cached, persona is frozen.

Call `resetTextSession(parts)` to delete the runtime.
Only then does the next call to `getTextSession(parts)` trigger a fresh
`buildRuntime()` → `loadPersonaContext(dir)` → current file contents.

`resetTextSessionForTests()` clears the entire `runtimes` map for test
isolation.

## Related

- [../identity/persona-loading.md](../identity/persona-loading.md) — the
  caching mechanism
- [../memory/docs-ingestion.md](../memory/docs-ingestion.md) — the ingestion
  pipeline
- [../runtime/turn-lifecycle.md](../runtime/turn-lifecycle.md) — the session
  model