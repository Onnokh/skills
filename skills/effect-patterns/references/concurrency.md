# Concurrency and structured concurrency

Effect's headline feature is *structured* concurrency: every fiber has a parent
scope, and when that scope ends, its fibers are interrupted and their finalizers
run. The whole discipline below follows from one rule — **never spawn work that
outlives the thing that owns it.**

## Forking: always into a scope

Don't reach for a bare "fire and forget". Tie every forked fiber to a scope so it
can't leak:

```ts
// Background work that should live as long as the surrounding scope:
yield* loop.pipe(Effect.forkScoped)

// Fork into a specific, explicitly-held scope:
const fiber = yield* work.pipe(Effect.forkIn(scope))

// Fork as a child of the current fiber (parent awaits/interrupts it):
const child = yield* work.pipe(Effect.forkChild)
```

`forkScoped` / `forkIn(scope)` are the workhorses: the fiber is interrupted
automatically when the scope closes, so shutdown is orderly and there are no
dangling background tasks. A truly detached daemon is a deliberate, rare choice —
reach for the scoped variants by default. (Effect's API has moved toward
scope-bound forking precisely because bare `fork` made leaks too easy.)

## Shared state: pick the primitive by the coordination shape

Don't reach for a mutable closure variable. Effect has a primitive for each kind
of coordination, and the choice communicates intent:

| Need | Primitive |
| --- | --- |
| A mutable cell, atomic updates | `Ref` |
| A cell whose updates run an *effect* atomically (no races) | `SynchronizedRef` |
| A cell others can *subscribe* to changes on | `SubscriptionRef` |
| One-shot "I'll have a value later" (request → response across fibers) | `Deferred` |
| A buffered hand-off between producer and consumer | `Queue` |
| Broadcast one event to many subscribers | `PubSub` |
| Limit how many fibers do X at once / mutual exclusion | `Semaphore` |

```ts
// Deferred: one fiber blocks until another completes it
const done = yield* Deferred.make<Result, MyError>()
// ...consumer: yield* Deferred.await(done)
// ...producer: yield* Deferred.done(done, exit)

// Semaphore as a per-key mutex — serialize writes to one resource
const sem = Semaphore.makeUnsafe(1)
yield* Semaphore.withPermit(sem)(criticalSection)   // permits = 1 → mutual exclusion
```

A per-key semaphore map (one permit per resource id) gives fine-grained locking
without a single global lock — far better throughput than serializing everything.

## Bound concurrency explicitly

`Effect.all` / `Effect.forEach` run sequentially unless you say otherwise. State
the concurrency, and choose the number for the workload — unbounded fan-out is how
you exhaust file handles, sockets, or rate limits:

```ts
// I/O-bound: cap parallelism so you don't open 500 sockets at once
yield* Effect.forEach(urls, fetchOne, { concurrency: 8 })

// Independent, cheap, and you want them all at once:
yield* Effect.all(tasks, { concurrency: "unbounded" })

// Fan out, discard results (side effects only):
yield* Effect.all(effects, { concurrency: "unbounded", discard: true })
```

`Effect.race(a, b)` gives "first to finish wins, the loser is interrupted" — the
idiom for a reader/writer pair on a connection, or a manual timeout.

## Interruption and cleanup are first-class

Because interruption is built in, cleanup must be attached to the effect, not
written at call sites. Three tools:

```ts
// Runs on success, failure, OR interruption — the "finally" of Effect:
work.pipe(Effect.ensuring(releaseResource))

// Register a finalizer on the current scope:
yield* Effect.addFinalizer((exit) => cleanup(exit))

// Protect a critical section from being interrupted mid-way:
yield* Effect.uninterruptible(commitThenAck)
```

Distinguish *interruption* from *failure* when reacting to a cause: interruption
is cancellation (usually re-raise it, don't treat it as an error). `Cause`
exposes this — see [error-recovery.md](error-recovery.md) for `Cause` inspection.

## Background daemons: repeat on a schedule, forked into scope

Long-lived periodic work (a cleanup loop, a poller) is the composition of three
things — a body, a `Schedule`, and a scoped fork — and it belongs in the layer
that owns the lifecycle, *not* in a method:

```ts
yield* cleanup.pipe(
  Effect.catchCause((c) => Effect.logError("cleanup failed", { cause: Cause.pretty(c) })),
  Effect.repeat(Schedule.spaced("1 hour")),
  Effect.forkScoped,                 // dies with the scope
)
```

Note the `catchCause` *inside* the loop: a daemon must survive its own failures,
so recover per-iteration rather than letting one error kill the loop. (`Schedule`
also drives *retries* — that's a different use, covered in
[error-recovery.md](error-recovery.md).)
