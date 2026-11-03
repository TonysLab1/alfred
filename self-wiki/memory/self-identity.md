# Self-Identity Resolution — Which Entity Is "Me"

When Alfred writes a conversational memory like "my boss Sarah works at NASA,"
it needs to know which Person entity is the speaker. `self-identity.ts` (341
lines) is the resolver that answers "who am I talking to."

## The marker

An OIP Entity is marked as self by setting `alfred.isSelf = true` on the
revision. The resolver scans all Entity records for this flag:

```ts
export async function resolveSelfEntity(provider): Promise<{id, name} | null>
```

It iterates every Entity (up to 8,000), reads the current revision, and checks
for `alfred.isSelf === true`. The first match wins. If no entity is marked, it
returns `null`.

## Auto-resolution when no self exists

`resolveOrCreateSelfEntity(provider, preferredName, now, upsertPerson)` handles
the common case where no entity has been explicitly marked. In order:

1. Check for an explicit self (the `isSelf` flag)
2. Try to find a Person matching `preferredName` (the `BRIEFING_USER_NAME` or
   derived display name)
3. Among matches, prefer one whose description contains "primary user"
4. Fall back to the first matching Person by name
5. Fall back to creating a new Person entity with the preferred name and summary
   "The user Alfred serves"

This means self-identity bootstraps lazily — Alfred starts serving a user
without any explicit configuration, and `setSelfPerson` retrospectively marks
the correct entity.

## setSelfPerson — the explicit path

`setSelfPerson(provider, personId)` marks a specific Person as self and
handles the fallout:

- Clears `isSelf` from any previous self entity
- Supersedes the generic "User" placeholder if one exists (marks it with
  `alfred.supersededBy` and `alfred.isSelf: false`)
- Copies personal fields (email, telephone, birthDate) from the placeholder or
  previous self onto the new self if the new self is missing them
- Migrates assertions that pointed at the previous self or placeholder to point
  at the new self
- Skips migration if an equivalent assertion already exists on the new self

The migration logic is `migrateAssertionsToSelf`: for every Assertion where
`subject` or `object` is the old self DID, it rewrites the DID to the new self.
If a duplicate assertion already exists (same subject + predicate + object
pointing at the new self), the old one is superseded rather than migrated.

## Where this is called

`resolveSelfEntity` is imported by `conversation-memory.ts` for
`writeConversationalMemory` — so that when an assertion about "me" is written,
the subject DID is the correct self entity.

`setSelfPerson` is callable from the memory inspection API and is a one-time
operation per profile — once a Person is marked as self, the resolver finds it
on every subsequent call.

## The bootstrapping story

A fresh Alfred profile with no person entities:

1. First user utterance triggers `commitTurn` → `extractConversationalMemory`
2. If the text names someone, the extractor produces a Person entity
3. That entity has no `isSelf` flag
4. Later, the user says "remember that I work at NASA"
5. The extractor produces an Assertion with `subjectName = "I"` or the display
   name
6. `writeConversationalMemory` calls `resolveOrCreateSelfEntity`, which either
   finds an existing Person matching the name or creates one
7. If the user hasn't set a self, the auto-resolution runs every time

The system works without explicit configuration, but assertions may be attached
to a placeholder "User" entity until `setSelfPerson` is called to consolidate
them onto the real Person record.

## Related

- [conversation-extraction.md](conversation-extraction.md) — the extraction
  pipeline that calls this
- [conversation-capture.md](conversation-capture.md) — where commitTurn
  triggers extraction
- [oip/record-model.md](oip/record-model.md) — the `alfred.isSelf` field on
  Entity records