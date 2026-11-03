# Extraction Contract — The Zod Schema Governing Structured Memory Output

Every LLM that extracts structured memory from text — whether during document
ingestion or during conversation — outputs JSON that must pass
`MemoryExtractionResultSchema` in
`packages/memory/src/oip-local/extraction-contract.ts`.

The contract is intentionally permissive at the top level (every array defaults
to `[]`) and strict in its design rule: **"Freeform model prose must never
become canonical memory directly."**

## The full schema

```ts
MemoryExtractionResultSchema = z.object({
  entities:             z.array(z.record(z.unknown())).default([]),
  episodes:             z.array(z.record(z.unknown())).default([]),
  observations:         z.array(z.record(z.unknown())).default([]),
  assertions:           z.array(z.record(z.unknown())).default([]),
  relationships:        z.array(z.record(z.unknown())).default([]),
  temporalReferences:   z.array(z.record(z.unknown())).default([]),
  scheduledSurfacing:   z.object({…}).default({}),
  publicKnowledge:      z.object({…}).default({}),
  memoryImportance:     z.number().min(0).max(1).optional(),
  ambiguities:          z.array(z.record(z.unknown())).default([]),
  needsResolution:      z.array(z.record(z.unknown())).default([]),
});
```

## What each field means

| Field | Purpose | Populated by |
|---|---|---|
| `entities` | Person, Organization, Thing records | Docs extraction, conversation extraction |
| `episodes` | Dated events with participants | Not currently produced |
| `observations` | Free-text observations linked to source | Document section ingestion |
| `assertions` | Subject-predicate-object statements | Docs extraction, conversation extraction |
| `relationships` | Edge definitions (source, predicate, target) | Not currently populated |
| `temporalReferences` | Time anchors linking records to dates | Not currently populated |
| `scheduledSurfacing` | Reminder metadata (remindAt, status, reason) | Audio note extraction |
| `publicKnowledge` | Public-source flagging for cross-instance sharing | Not currently used |
| `memoryImportance` | 0–1 relevance score for retrieval weighting | Not currently used |
| `ambiguities` | Things the extractor is uncertain about | Signalled but not acted on |
| `needsResolution` | Things that need follow-up before they become records | Signalled but not acted on |

## The permissive vs. strict tension

The array fields are `z.array(z.record(z.unknown()))` — the schema accepts any
JSON object in any array position. It does NOT validate that `entities` actually
contain `tempId` strings, or that `assertions` actually carry `subjectTempId`,
`predicate`, and `object`.

This is deliberate: the schema validates *structural shape* (is it an array of
objects?) but defers *semantic validation* to the extraction prompt and the
downstream writer. The contract's job is to prevent total garbage from landing
in OIP, not to guarantee perfectly formed records.

The extraction prompt in `docs-ingest/extract.ts` supplies the expected shape
(`entities: [{tempId, name, entityClass, summary, quote}]`) but the schema does
not enforce it. This means invalid records (missing tempId, missing predicate)
are possible if the prompt is changed or the model drifts.

## Where this contract is consumed

- `packages/memory/src/docs-ingest/extract.ts` — `parseExtractionJson()` wraps
  the LLM response with `safeParse`, silently returning `emptyExtraction()` on
  failure
- `packages/memory/src/oip-local/provider.ts` — `commitTurn` calls
  `extractConversationalMemory` which uses heuristic extraction, not LLM
  extraction; the contract is not invoked in the conversation path

So the contract is enforced in the **document ingestion pipeline** but NOT in
the **conversation extraction path.** The conversation path uses regex
heuristics, not LLM output, and therefore the contract doesn't apply to it.

## What ambiguities and needsResolution are for

These are signalling fields that exist in the schema but have no downstream
consumer. An extraction can declare "I'm unsure about this entity's name" or
"this assertion needs the user to confirm before it enters memory" — but nothing
currently reads those fields and acts on them.

They represent an intentional gap: the contract supports declaring uncertainty,
the pipeline that would act on uncertainty is not yet built.

## Related

- [conversation-extraction.md](conversation-extraction.md) — the heuristic path
  that does NOT use this contract
- [docs-ingestion.md](docs-ingestion.md) — the ingestion path that DOES
- [oip/record-model.md](oip/record-model.md) — the records the extraction
  produces