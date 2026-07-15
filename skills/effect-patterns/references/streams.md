# Streams

`Stream<A, E, R>` is an effectful source that emits many `A` values over time,
fails with `E`, and requires services `R`. Streams are pull-based and
backpressured — consumption controls demand. Use them for sources that are
naturally many-valued and time-ordered: gateway events, provider callbacks
adapted through queues, event logs, paginated APIs, file/stdin sources, and
pipelines that filter/map/buffer/throttle.

**Don't use a stream just to loop forever.** One repeated effect with no
emitted values is `Effect.repeat(...)` with a `Schedule` — see the daemon
pattern in [concurrency.md](concurrency.md).

## Choosers

Pick by shape rather than memorizing the API surface:

**Sources**

| Source | Constructor |
| --- | --- |
| In-memory values / test fixtures | `Stream.make(...)`, `Stream.fromIterable(...)` (+ `Stream.concat(Stream.never)` for an open subscription in tests) |
| Queue-backed callback boundary | `Queue` + `Stream.fromQueue(...)` |
| Broadcast events | `PubSub` + `Stream.fromPubSub(...)` |
| Latest value + updates | `SubscriptionRef` |
| Schedule-generated ticks | `Stream.fromSchedule(...)` |
| Paginated pull API | `Stream.paginate(...)` — its step function is already effectful (`Effect<[chunk, Option<nextState>]>`); there is no separate `paginateEffect` |
| Async iterable / platform source | `Stream.fromAsyncIterable(...)` |
| Stream built after reading services/config | `Stream.unwrap(...)` |

**Transformations**: `Stream.map` (pure) / `Stream.mapEffect` (effectful, takes
`{ concurrency }`, plus `unordered: true` when order is irrelevant and latency
matters) / `Stream.flatMap` (one-to-many, takes `{ concurrency }`) /
`Stream.filter` / `Stream.mapAccum` (stateful).

**Consumption**: `Stream.runForEach` (side-effecting consumer),
`Stream.runDrain` (run, ignore elements), `Stream.runFold` (reduce),
`Stream.runCollect` (tests and small finite streams **only** — never on
unbounded production streams; in tests bound it with `Stream.take(n)` first).

## Long-lived consumers live in layers

A long-lived stream consumer is owned by a layer and forked into the layer
scope, exactly like the daemons in [concurrency.md](concurrency.md) — the layer
owns the lifetime, and shutdown interrupts the consumer:

```ts
export const layer = Layer.effectDiscard(
  Effect.gen(function* () {
    const gateway = yield* Gateway.Service
    yield* gateway.events.pipe(
      Stream.filter(isMessageEvent),
      Stream.runForEach(handleEvent),
      Effect.forkScoped,
    )
  }),
)
```

If service *methods* need to fork work into the layer's lifetime, capture
`Scope.Scope` during layer acquisition and use `Effect.forkIn(scope)`
internally — don't expose the scope as public API.

## Expose streams, hide producers

Service interfaces should expose a `Stream` when callers consume events; the
producer `Queue`/`SubscriptionRef` stays private to the implementation:

```ts
export interface Interface {
  readonly events: Stream.Stream<ProviderEvent, ProviderError>
  readonly status: Stream.Stream<ProviderStatus>
}
```

Pick the producer by delivery semantics: `Queue` when each item goes to one
consumer, `PubSub` when every subscriber sees every event, `SubscriptionRef`
when consumers need current-value-plus-changes.

## Backpressure and buffers

Natural pull-based backpressure is the default — reach for `Stream.buffer(...)`
only when producer and consumer should genuinely decouple, and choose the
strategy by what overflow should mean: `"suspend"` (backpressure), `"dropping"`
(drop new), `"sliding"` (keep latest). `capacity: "unbounded"` is rare and only
safe when growth is bounded elsewhere. `Stream.debounce` for quiet-period
behavior; `Stream.throttle` for rate-shaping.

## Errors

Same failure/defect discipline as [error-recovery.md](error-recovery.md):
prefer typed stream errors, translate at boundaries with `Stream.mapError`,
recover typed cases with `Stream.catchTag`/`catchIf`/`catchFilter`, and reserve
`Stream.catchCause` for explicit supervision boundaries. Don't hide stream
defects by default — let them reach the owning layer unless the stream is
explicitly best-effort.

## Keyed concurrency

For streams of work keyed by session/channel/id where each key needs ordered
processing but different keys can run in parallel: build **one named helper**
on `FiberMap` (e.g. a `runForEachKeyed` that serializes per key and coalesces
pending values into a latest-value rerun) instead of scattering ad-hoc maps of
fibers through consumers.
