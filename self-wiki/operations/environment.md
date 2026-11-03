# Environment Variables

The `.env` file at the repository root controls Alfred's runtime behaviour.
This document catalogs the variables that affect self-knowledge and memory.

## Identity and persona

| Variable | Effect | Default |
|---|---|---|
| `ALFRED_PROFILE_ID` | Which profile directory to use for persona and memory | `profile.default` |
| `ALFRED_PERSONA_DIR` | Override persona directory path | `./data/persona/<profileId>/` |
| `ALFRED_ARCHIVE_URL` | Enable HYPATIA archive persona override | (unset) |

`ALFRED_PERSONA_DIR` takes priority over the profile-derived path. When set,
`defaultPersonaDir()` returns the resolved value of this variable. Documented
in `.env` (commented out) with an example pointing to `alfred-home/backend`.

## Memory

| Variable | Effect | Default |
|---|---|---|
| `ALFRED_MEMORY_PROVIDER_ID` | Active memory provider for retrieval and commitTurn | `memory.oip-local` |
| `ALFRED_MEMORY_OIP_PATH` | Root directory for OIP store | `./data/memory-oip/<profileId>/` |
| `ALFRED_MEMORY_LOCAL_PATH` | Path for JSONL memory provider | `./data/memory/<profileId>.jsonl` |
| `ALFRED_X_INGEST_DIR` | Root for X ingest sources and ledger | `./data/memory/<profileId>/ingest` |

## LLM

| Variable | Effect |
|---|---|
| `OPENAI_API_KEY` | Required for LLM extraction during docs ingestion |
| `OPENAI_DOCS_EXTRACT_MODEL` | Model for docs extraction (default: `gpt-5.6-terra`) |
| `GROK_API_KEY` | Alternative LLM provider for conversation |
| `ALFRED_LLM_PRESET` | Preset configuration for LLM selection |

## Voice

| Variable | Effect |
|---|---|
| `DEEPGRAM_API_KEY` | Deepgram STT provider |
| `ELEVENLABS_API_KEY` | ElevenLabs TTS provider |
| `ELEVENLABS_VOICE_ID` | Specific voice to use |
| `LIVEKIT_API_KEY` / `LIVEKIT_API_SECRET` | LiveKit room credentials |
| `LIVEKIT_URL` / `LIVEKIT_ROOM` / `LIVEKIT_IDENTITY` | LiveKit connection details |

## Desktop and sidecar

| Variable | Effect | Default |
|---|---|---|
| `PORT` | HTTP port for desktop client | `3000` |
| `ALFRED_SIDECAR_MODE` | Run desktop client as HTTP API only | `false` |
| `ALFRD_CLOUD_DISABLED` | Disable cloud sync (`alfrd.net`) | `false` |

## Briefing

| Variable | Effect |
|---|---|
| `BRIEFING_TIMEZONE` | Timezone for scheduling and reminders |
| `BRIEFING_DAY_START` | Time of day briefing becomes available (HH:MM) |

## Testing

| Variable | Effect |
|---|---|
| `NODE_ENV` | Set to `test` for test isolation |
| `ALFRED_LOG_PLAYBACK` / `ALFRED_LOG_STT` / `ALFRED_LOG_VOICE` | Debug logging flags |

## Related

- [repository-layout.md](repository-layout.md) — where these point
- [../identity/persona-override.md](../identity/persona-override.md) —
  `ALFRED_ARCHIVE_URL` effect
- [../memory/docs-ingestion.md](../memory/docs-ingestion.md) — docs extraction
  depends on `OPENAI_API_KEY`