# Error recovery and resilience

This is the *handling* side of errors — retrying, timing out, recovering,
degrading. (For *defining* errors as tagged schema classes, see
[errors.md](errors.md).) The throughline: Effect separates **failures** (expected,
in the typed error channel) from **defects** (bugs, in the cause), and good
resilience code treats them differently at every step.

## Retry with a `Schedule`

Don't hand-roll retry loops. A `Schedule` is a composable, reusable retry policy:

```ts
// Exponential backoff with jitter, capped attempts — the standard for flaky I/O:
work.pipe(Effect.retry({
  times: 3,
  schedule: Schedule.exponential("200 millis").pipe(Schedule.jittered),
}))
```

Compose the policy from orthogonal pieces: `Schedule.exponential` /
`Schedule.spaced` (timing) `.pipe(Schedule.jittered)` (anti-thundering-herd)
combined with `Schedule.recurs(n)` or a `while`/`until` predicate (termination).
**Jitter is composed in, not baked in** — that separation is the point.

For policies that must react to the *error* (read a `Retry-After` header, stop on
non-retryable status codes, surface attempt count to a UI), build a stateful
schedule that inspects each error and decides delay-or-stop. The transferable
idea: **retry logic is data (a `Schedule`), so it can encode arbitrary
error-driven backoff without imperative loops.**

(A `Schedule` also drives *recurring* background work — see
[concurrency.md](concurrency.md). Same abstraction, different verb: `retry` on
failure vs `repeat` on success.)

Semantics worth knowing cold, because off-by-one retry bugs are silent:

- The source effect runs **once before the schedule is stepped** —
  `Schedule.recurs(3)` means three retries *after* the initial run (4 attempts).
- `Effect.retry` retries typed **failures** only; defects and interruption are
  never retried.
- `Effect.repeat` repeats **successes**; a failure stops the loop unless the
  body handles it first (see the polling-worker shape below).
- `Schedule.spaced` waits after the work completes; `Schedule.fixed` aligns to
  a cadence regardless of how long the work took.
- `Effect.retryOrElse(policy, onExhausted)` when exhausted retries need a
  fallback or reporting effect; `Schedule.tapInput` to log each retried error.
- Retry only at the narrowest boundary, and only operations with **proven
  idempotency**. Exhausted failures should stay visible unless the boundary has
  a truthful fallback.

### Rate-limit-aware retry

When a typed error carries a provider backoff hint (`retryAfterMs` — see the
reason-union getters in [errors.md](errors.md)), let the schedule take the
larger of its own backoff and the provider's delay:

```ts
const providerRetrySchedule: Schedule.Schedule<RateLimited, RateLimited> =
  Schedule.exponential("200 millis").pipe(
    Schedule.jittered,
    Schedule.upTo({ times: 5 }),
    Schedule.passthrough,             // feed the error through to modifyDelay
    Schedule.modifyDelay(({ input, duration }) =>
      Effect.succeed(
        input.retryAfterMs === undefined
          ? duration
          : Duration.max(duration, Duration.millis(input.retryAfterMs)),
      ),
    ),
  )
```

For HTTP specifically, `HttpClient.retryTransient` / `withRateLimiter` cover
the generic tiers — see [http-clients.md](http-clients.md).

## Polling workers and per-item isolation

A polling worker's pass should handle its *expected* typed failures before the
`repeat`, so one bad pass logs and continues while genuine defects still
escalate to supervision:

```ts
const pass = runPass().pipe(
  Effect.tapError((error) => Effect.logError("Worker.pass_failed", error)),
  Effect.ignore,          // expected operational failures: log and continue
)
const run = pass.pipe(Effect.repeat(Schedule.spaced("1 second")))
```

Same idea inside a batch: catch expected item-level failures around each item
so one bad item doesn't stall the rest — but only when skipping or retrying
the item later is truthful product policy:

```ts
yield* Effect.forEach(
  items,
  (item) =>
    processItem(item).pipe(
      Effect.tapError((e) =>
        Effect.logError("Worker.item_failed", e).pipe(
          Effect.annotateLogs({ itemId: item.id }),
        ),
      ),
      Effect.ignore,
    ),
  { discard: true, concurrency: 5 },
)
```

Prefer these typed-failure shapes over cause-level recovery; reserve
`catchCause` for supervision boundaries whose policy really is "report any
non-interrupt failure and continue" (and even there, exclude interruption —
see below).

## Timeouts: failure vs alternative

Two variants, and the choice encodes intent:

```ts
// Timeout becomes a catchable TimeoutError in the typed channel:
work.pipe(Effect.timeout("5 seconds"), Effect.catchTag("TimeoutError", () => fallback))

// Timeout diverts to an alternative effect (here, escalate to a defect):
work.pipe(Effect.timeoutOrElse({ duration: "5 seconds", orElse: () => Effect.die("timed out") }))
```

Use `timeout` when a timeout is an *expected, recoverable* outcome; `timeoutOrElse`
when you want to branch (including escalating to `die` for "this should never take
this long").

## Catching: recover, reframe, or rethrow

Match the catch combinator to how precisely you want to discriminate:

| Combinator | Use for |
| --- | --- |
| `catchTag` / `catchTags` | recover specific tagged errors (the common case) |
| `catchIf(predicate, …)` | recover only when a type-guard/condition matches |
| `catchAll` | collapse the whole failure channel into a recovery |
| `catchCause` | inspect the *full* cause — needed when code may sync-throw (defects) |
| `orElse` / `orElseSucceed` | "try this, else that" / "else this default value" |
| `orDie` | turn a recoverable failure into a fatal defect |

```ts
// Reframe one error into another (e.g. a domain error → an API error):
svc.get(id).pipe(Effect.catchTag("NotFoundError", () => Effect.succeed(undefined)))

// Recover only an expected subset, by predicate:
svc.get(id).pipe(Effect.catchIf(NotFoundError.is, () => Effect.succeed([])))
```

A subtle but important one: when the code under a handler can throw *synchronously*
(JSON/JSONC parsing, schema decode), use `catchCause`, not `catchTag`/`orElse` —
sync throws arrive as **defects**, which the failure-channel combinators won't
catch. `Cause.squash(cause)` then extracts the underlying error for logging.

## Failure vs defect, and inspecting the cause

The distinction drives recovery decisions:

- **Failure** = expected, typed, recoverable. Belongs in the error channel.
- **Defect** = a bug or an unexpected throw. Belongs in the cause; usually you log
  it and return a generic 500, rather than handling it semantically.
- `orDie` *promotes* a failure to a defect ("this error is fatal here, stop
  pretending it's recoverable") — good for must-succeed I/O at a boundary.

Tools for inspecting outcomes without throwing:

```ts
const exit = yield* work.pipe(Effect.exit)        // Exit<A, E> — examine, log, decide
const either = yield* work.pipe(Effect.either)    // Either<E, A> — branch in pure code
const maybe = yield* work.pipe(Effect.option)     // Option<A> — failure → None
```

And always separate interruption from real failure in a `catchCause`:

```ts
work.pipe(Effect.catchCause((cause) => {
  if (Cause.hasInterruptsOnly(cause)) return Effect.interrupt   // cancellation — re-raise
  return Effect.succeed(degraded(Cause.squash(cause)))          // genuine failure — degrade
}))
```

## Graceful degradation

Decide per call site how much a failure should cost. The vocabulary:

```ts
readConfig.pipe(Effect.orElseSucceed(() => ({})))            // missing → empty default
readFile.pipe(Effect.catch(() => Effect.succeed("")))         // failure → degraded value
fireAndForget.pipe(Effect.ignore, Effect.forkScoped)          // don't care about outcome
response.stream.pipe(Stream.catchCause(() => Stream.empty))   // stream dies → end cleanly
```

The skill is matching severity to consequence: a broken global config → fall back
to defaults and log; a critical write → `orDie`; a best-effort notification →
`ignore`.
