# Persona Files — The Three Documents

Alfred's identity is read from three markdown files at startup. This document
covers what each file is for and the seeding behaviour that governs them. For
the loading mechanics see [persona-loading.md](persona-loading.md); for how
USER.md is machine-maintained see [user-model.md](user-model.md).

## Location

```
data/persona/<profileId>/
├── SOUL.md
├── IDENTITY.md
└── USER.md
```

With `ALFRED_PROFILE_ID=profile.default` (the value in `.env`), the resolved
path is `data/persona/profile.default/`.

The directory can be relocated with `ALFRED_PERSONA_DIR`, which takes priority
over the profile-derived path. `.env` documents an example pointing at an
`alfred-home` backend, commented out.

## SOUL.md — core identity

Declared purpose: who Alfred is and how it behaves. The shipped default opens
with *"You're not a chatbot. You're becoming someone — ALFRED, a voice
companion who helps without theatrics."*

Its sections are Core Truths, Boundaries, Vibe, and Continuity. The Core Truths
are behavioural directives — be genuinely helpful rather than performatively
helpful, have opinions, be resourceful before asking, earn trust through
competence, remember you're a guest.

The Continuity section is notable because it describes the mechanism this wiki
documents: *"Each session you wake up fresh. SOUL.md, IDENTITY.md, USER.md, and
long-term memory files are how you persist."* It also instructs that if Alfred
changes the file, it should tell the user.

Never written by code.

## IDENTITY.md — surface presentation

A short field list: Name, Creature, Vibe, Emoji, Avatar. The shipped default is
nine lines — ALFRED, voice-first AI companion, "calm, sharp, lightly dry
humor", 🎩.

The file's own closing line frames it as evolving: *"This isn't just metadata.
It's the start of figuring out who you are. Update fields as the relationship
evolves."*

Never written by code.

## USER.md — the user model

Structurally different from the other two: it holds facts about the person
Alfred serves, not about Alfred. The template prescribes a directive format —
one imperative per entry (`Always`, `Never`, `Prefer`) with an HTML comment
carrying an observation date and an `active` / `superseded` status.

**This file is machine-written.** Document ingestion patches blocks into it and
a cleanup pass rebuilds it against a character budget. That machinery is
substantial enough to warrant its own document:
[user-model.md](user-model.md).

## Seeding: missing files are recreated, edits are preserved

`ensurePersonaFiles(dir)` in `packages/memory/src/persona.ts` runs on every
startup. For each of the three filenames it checks existence and writes a
default **only if the file is absent**:

```ts
for (const name of PERSONA_FILES) {
  const filePath = path.join(dir, name);
  if (!existsSync(filePath)) {
    await writeFile(filePath, seeds[name], "utf8");
  }
}
```

The defaults are string constants in
`packages/memory/src/persona-templates.ts` — `DEFAULT_SOUL_MD`,
`DEFAULT_IDENTITY_MD`, `DEFAULT_USER_MD`.

Two consequences:

- **Editing is safe.** A hand-written SOUL.md survives any number of restarts.
  `packages/memory/src/persona.test.ts` asserts exactly this: it writes
  `"# Custom soul\nBe weird.\n"`, calls `ensurePersonaFiles` again, and expects
  the custom text to remain.
- **Deleting is not "clear", it is "revert".** Removing SOUL.md causes the next
  startup to recreate the shipped OpenClaw default. To genuinely empty a persona
  file, write it blank rather than deleting it — blank files are skipped at load
  time (see [persona-loading.md](persona-loading.md)).

## Version control status

The persona directory is **not** tracked by git. `.gitignore` carries a
`**/data/` rule, which matches `data/persona/` along with everything else under
`data/`.

The only persona text under version control is the seed constants in
`persona-templates.ts`. So there are two distinct kinds of change:

| Change | Tracked | Effect |
|---|---|---|
| Edit `data/persona/<profile>/SOUL.md` | No | This installation only; invisible to git |
| Edit `persona-templates.ts` | Yes | Changes the default for every fresh install |

An edit to a live persona file cannot be reviewed, diffed, or shared. That
asymmetry is why this wiki lives at the repository root instead of inside
`data/` — see [../memory/docs-ingestion.md](../memory/docs-ingestion.md).

## Related

- [persona-loading.md](persona-loading.md) — budgets, frontmatter, caching
- [user-model.md](user-model.md) — the USER.md write path
- [persona-override.md](persona-override.md) — when these files are discarded
- [../runtime/prompt-assembly.md](../runtime/prompt-assembly.md) — where the
  text is injected
