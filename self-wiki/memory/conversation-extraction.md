# Conversational Memory Extraction — The Regex Pipeline That Runs Every Turn

Inside every `commitTurn`, after the Observation is written, the provider runs
a regex/heuristic parser over the turn text. This is
`extractConversationalMemory()` in `packages/memory/src/conversation-memory.ts`
— a 200+ line function with no LLM dependencies, just pattern matching.

## What it extracts

The parser produces a `ConversationalMemoryWrite`:

```ts
interface ConversationalMemoryWrite {
  entities?: ConversationalEntitySpec[];
  assertions?: ConversationalAssertionSpec[];
  notes?: string[];
}
```

Entities carry: `name`, `entityClass` (Person or Organization or Thing),
`summary`, optional `email`, `telephone`, `birthDate`.

Assertions carry: `subjectName`, `predicate`, `objectName`, optional `text`.

If the extraction yields structured content, `writeConversationalMemory`
creates OIP packages for each entity and assertion, wiring them into the graph
via drefs.

## The filtering layers

The parser applies multiple passes to avoid creating junk entities from
misheard speech. These are real compensation mechanisms for STT failures.

### FALSE_PERSON

A 60-word blocklist preventing common words from being treated as person names:
`a`, `the`, `my`, `his`, `her`, `boss`, `supervisor`, `manager`, `coworker`,
`wife`, `husband`, `son`, `daughter`, `mom`, `dad`, `alfred`, `albert`,
`person`, `someone`, `guy`, `gal`, `folks`, `that`, `who`, `whom`, `named`,
`specialist`, `recruiter`, `at`, `for`, `from`, `to`, `in`, `on`, `of`,
`is`, `are`, `was`, `were`.

### PLACEHOLDER_PERSON

Catches phrases like "person that," "someone who," "people that," "guy who" —
STT artifacts that parse as names but aren't.

### ORG_ACRONYMS

A set of 24 known-acronym organizations that STT often lowercases: USPTO, NASA,
IBM, OPM, FBI, CIA, NSA, IRS, DOJ, DOD, VA, EPA, FDA, NIH, NSF, SEC, FTC,
FCC, MIT, CMU, NYU, UCLA, USC. These survive the cleaner when STT output is
lowercase.

### ORG_STOP_TRAIL

Trims trailing speech after organization names: "as," "and," "but," "where,"
"because," "since," "starting," "from," "now," "currently," "today,"
"recently," etc. — all the ways a sentence continues after naming an org.

### FALSE_ORG

Prevents generic nouns from being treated as organization names: "the," "my,"
"work," "job," "company," "office," "place," "home," "there," "here."

### WORKPLACE_ROLE / WORKPLACE_CONTACT_ROLE

Recognizes employment relationships: "boss," "supervisor," "manager,"
"coworker," "colleague," "teammate," plus HR-specific titles like
"HR specialist," "HR contact," "recruiter," "onboarding contact," "POC."

### cleanPersonName / cleanOrgName

Remove possessives, trailing punctuation, and title-case the result (while
preserving known acronyms in uppercase).

## The pattern: archaeological accuracy

The acronym list, the FALSE_PERSON entries, and the ORG_STOP_TRAIL patterns are
not designed from first principles. They are accumulated corrections from
observed STT failures — real names that got lowercased to common words, real
organizations that speech didn't capitalise.

The extractor is therefore accurate about real-world STT behaviour and opaque
about why any given entry is present. The code can be described exactly ("this
list exists") and not at all ("why these 24 acronyms"). This is not a defect in
the code or the documentation — it is the nature of a heuristic compensator
trained on live data.

## Where extraction output goes

`writeConversationalMemory` in the same file takes the extraction result and
writes structured OIP records. `self-identity.ts` resolves which Entity is the
current user, so assertions about "me" or "my boss" can be linked correctly.

The extraction runs on BOTH user turns and assistant turns, because
`commitTurn` runs for both roles. An assistant reply that says "your boss Sarah
works at NASA" produces the same Person and Organization entities as the user
saying "my boss Sarah works at NASA."

## Why this isn't in the prompt assembly doc

Extraction is a side effect of memory commit, not a prompt-prep step. The LLM
never sees the extracted entities or assertions during the turn that produced
them — they are written to OIP and become retrievable on later turns.

## Related

- [conversation-capture.md](conversation-capture.md) — the commitTurn path that
  calls this
- [self-identity.md](self-identity.md) — how "me" is resolved
- [oip/record-model.md](oip/record-model.md) — the records this produces
- [retrieval.md](retrieval.md) — how these records are later found