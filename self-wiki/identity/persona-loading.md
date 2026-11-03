# Persona Loading — Path Resolution, Budgets, and Caching

How the three persona files get from disk into a prompt. Implementation lives
in `packages/memory/src/persona.ts` (89 lines) with the type contract in
`packages/contracts/src/prompt.ts`.

## The closed schema

Two hardcoded declarations define the entire persona surface.

In `packages/memory/src/persona.ts:14`:

```ts
export const PERSONA_FILES = ["SOUL.md", "IDENTITY.md", "USER.md"] as const;
```

In `packages/contracts/src/prompt.ts:16`:

```ts
export const PersonaContextSchema = z.object({
  soul: z.string().optional(),
  identity: z.string().optional(),
  user: z.string().optional(),
  dir: z.string().optional(),
});
```

**Adding a markdown file to the persona directory has no effect.** Nothing
enumerates the directory; the loader reads exactly three known filenames into
exactly three known fields. A `POLICIES.md` or `TERRAIN.md` dropped in beside
SOUL.md is inert — not an error, simply never read.

Extending the set means touching, at minimum: `PERSONA_FILES`,
`PersonaContextSchema`, `loadPersonaContext()`, `ensurePersonaFiles()`'s seed
map, both prompt injection sites (see
[../runtime/prompt-assembly.md](../runtime/prompt-assembly.md)), and two tests
that assert the current three-file contract.

This closed schema is the direct reason self-knowledge is delivered through the
memory graph instead of through more persona files — see
[layered-self-knowledge.md](layered-self-knowledge.md).

## Path resolution

`defaultPersonaDir(profileId)` resolves in two steps:

```ts
const fromEnv = process.env.ALFRED_PERSONA_DIR;
if (fromEnv && fromEnv.trim()) return path.resolve(fromEnv);
return path.join(resolveRepoRoot(), "data", "persona", profileId);
```

`ALFRED_PERSONA_DIR` wins when set. Otherwise the path is derived from the
profile id, which comes from `ALFRED_PROFILE_ID` (currently `profile.default`).

## Character budgets

Three constants cap how much persona text can reach the prompt:

| Constant | Value | Applies to |
|---|---|---|
| `MAX_CHARS_PER_FILE` | 20,000 | SOUL.md, IDENTITY.md |
| `USER_MD_MAX_CHARS` | 12,000 | USER.md |

`USER_MD_MAX_CHARS` is exported, because the USER.md cleanup machinery derives
its own targets from it — see [user-model.md](user-model.md).

Over-budget files are truncated rather than rejected, with a visible marker
appended:

```ts
return (
  trimmed.slice(0, maxChars) +
  `\n\n…[truncated ${trimmed.length - maxChars} chars; edit ${path.basename(filePath)} to shorten]`
);
```

Truncation is silent in logs but visible in the prompt itself. If Alfred's
persona appears to stop mid-sentence, this is why.

These are ceilings, not targets. Every character counts against **every turn of
every session**, permanently — persona text is a prompt prefix, not a retrieved
document.

## Blank files are skipped

`readPersonaFile()` returns `undefined` for a file that is missing, unreadable,
or whitespace-only:

```ts
const trimmed = stripFrontmatter(raw).trim();
if (!trimmed) return undefined;
```

Downstream, `PromptAssembler` only pushes a section when the field is truthy.
So a blank USER.md produces no `USER.md (user model):` block at all — not an
empty one. `persona.test.ts` asserts this by writing `"   \n"` to USER.md and
expecting `ctx.user` to be `undefined`.

**This is how to genuinely clear a persona file.** Deleting it triggers
re-seeding of the default; blanking it does not.

## YAML frontmatter is stripped

`stripFrontmatter()` removes a leading `---\n …  \n---\n` block:

```ts
function stripFrontmatter(raw: string): string {
  if (!raw.startsWith("---\n")) return raw;
  const end = raw.indexOf("\n---\n", 4);
  if (end === -1) return raw;
  return raw.slice(end + 5);
}
```

Only a block at the very start of the file, and only with `\n` line endings. A
file whose frontmatter is preceded by a blank line keeps its frontmatter as
literal prompt text.

## Loading is once per runtime, not once per turn

This is the operationally important behaviour and the easiest thing to get
wrong.

Two entry points exist:

- `ensureAndLoadPersona(profileId)` — seeds then loads. Used by the voice agent
  at `apps/voice-agent/src/brain.ts:223`.
- `loadPersonaContext(dir)` — loads only. Used by the desktop text path at
  `apps/desktop-client/src/lib/text-session.ts:189`.

Both are called during runtime construction. In the desktop path, runtimes are
memoised in a module-level `Map` keyed by session key:

```ts
export async function getTextSession(parts = {}) {
  const key = conversationSessionKey(parts);
  let runtime = runtimes.get(key);
  if (!runtime) {
    runtime = await buildRuntime(key, parts);
    runtimes.set(key, runtime);
  }
  …
}
```

`buildRuntime` is what loads persona. Once a runtime is cached, **persona is
frozen for the life of that runtime.**

### Consequence

Editing SOUL.md while Alfred is running changes nothing in the current session.
The next turn still uses the persona text read at construction time.

To pick up a persona edit:

- call `resetTextSession(parts)` — cancels the session and deletes the runtime
  from the map, so the next `getTextSession` rebuilds and reloads; or
- restart the process.

`resetTextSessionForTests()` clears the whole map and exists for test isolation.

This is the most likely cause of a false negative when evaluating a persona
change: edit the file, talk to the running instance, observe no difference, and
wrongly conclude the edit had no effect.

## Related

- [persona-files.md](persona-files.md) — what the three files hold
- [user-model.md](user-model.md) — the machine-written one
- [persona-override.md](persona-override.md) — a code path that discards all of
  this at runtime
- [../operations/change-and-verification.md](../operations/change-and-verification.md)
  — verification procedure including the reset step
