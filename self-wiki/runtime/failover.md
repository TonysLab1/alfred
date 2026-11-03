# Provider Failover — Stickiness, Cooldown, and Auto-Recovery

`StickyFailoverController` in `packages/core/src/failover.ts` governs how
Alfred handles provider failures. It is not a simple "try next" — it introduces
stickiness, cooldown periods, health probing, and automatic primary restoration
with a configurable interval.

Three separate controllers exist, one per modality (`stt`, `llm`, `tts`). Each
is constructed from a `ProviderPriorityList` and a `HealthProbe` callback.

## Construction

```ts
new StickyFailoverController(sessionId, list, clock, events, healthProbe)
```

`ProviderPriorityList` supplies:

- `orderedProviderIds` — priority-ordered list of provider IDs
- `settings` — `FailoverSettings`: thresholds, cooldown, retry interval, manual
  pin flags

The first provider in the list is designated the `primaryProviderId`. If the
settings carry a `manualPin`, that pinned provider becomes the active one
instead. The active provider is what the session actually uses.

## Key state variables

| Field | Meaning |
|---|---|
| `activeProviderId` | The provider currently in use |
| `primaryProviderId` | The first in the ordered list — what we want to use |
| `consecutiveFailures` | Per-provider failure counter (reset on success) |
| `cooldownUntil` | Per-provider timestamp — don't retry until after this |
| `stickySince` | When the failover happened — used for auto-restore timing |
| `lastFailoverAt` | Timestamp of the last failover event |

## Lifecycle

### Initial selection

`selectInitial()` health-probes each provider in order, skipping any in
cooldown, and picks the first that returns `healthy` or `degraded`. If the
first matching is not the primary, stickiness is not engaged yet — this is
just initial selection.

### Failure

`recordFailure(providerId, failureClass)` increments the failure counter for
the provider. If:

1. the failure class is **failover-eligible** (`FAILOVER_ELIGIBLE_FAILURES` from
   contracts), AND
2. the consecutive failure count meets or exceeds `consecutiveFailureThreshold`,
   AND
3. `manualPin` is not set

...then the controller:

- puts the failed provider in cooldown for `cooldownMs`
- scans forward through `orderedProviderIds` (wrapping past the end) for the
  next non-cooldown provider
- switches `activeProviderId` to that provider
- resets its failure counter to zero
- sets `stickySince` to now — this is the stickiness marker
- emits `provider.failover` with `{from, to, sticky: true}`

If the found next provider is the same as the current one, no switch occurs —
the controller stays on the failing provider rather than flip-flopping.

### Success

`recordSuccess(providerId)` resets the failure counter to zero. This means a
single success after a string of failures clears the count — the stickiness
relies on the `cooldownUntil` filter, not the failure counter accumulation.

### Auto-restore to primary

`maybeRestorePrimary()` checks three conditions:

1. `manualPin` is not set
2. the current active provider is not already the primary
3. enough time has passed since `stickySince`: `retryPrimaryIntervalMs`

If all three pass, it calls `checkPrimary({ bypassCooldown: false })`, which
health-probes the primary (respecting cooldown). If the primary is healthy:

- `activeProviderId` switches back to primary
- `stickySince` is cleared
- the primary's failure count and cooldown are reset
- `provider.primary_restored` event is emitted

### Manual/interval restore

`checkPrimary({ bypassCooldown = true })` is the user-facing or
scheduled-check path. `bypassCooldown` defaults to `true` for explicit user
requests ("check the primary now") but `false` for interval-driven checks (which
must respect cooldown).

The simulator scenarios 03–05 test this exact chain: failure → sticky failover →
user-triggered restore → interval-based auto-restore.

### Configuration update

`updateConfiguration(list)` replaces the priority list and settings at runtime.
It preserves the current `activeProviderId` if it still appears in the new list;
otherwise falls back to the new primary. `stickySince` is cleared on a new
active provider.

## What manualPin means

When `manualPin` is set with a `pinnedProviderId`, the controller:

- skips failover entirely — failures are recorded but never trigger a switch
- ignores auto-restore
- `selectInitial()` picks the pinned provider regardless of health

This is the "I know what I want, don't touch it" setting. The simulator 13/14
tests exercise this indirectly through the unified pipeline mode, which locks
all three selectors.

## Events emitted

| Event | Trigger |
|---|---|
| `provider.selected` | After selecting initial or switching |
| `provider.failed` | After any failure is recorded |
| `provider.failover` | After a sticky failover switch |
| `provider.health_probed` | After a health probe completes |
| `provider.primary_restored` | After primary is successfully restored |

## Related

- [state-machine.md](state-machine.md) — the `Failed` and `Recovering` states
  that failover interacts with
- [turn-lifecycle.md](turn-lifecycle.md) — where provider calls happen
- [../surfaces/simulator.md](../surfaces/simulator.md) — the scenarios that
  exercise failover exhaustively