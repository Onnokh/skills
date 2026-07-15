# Interop: meeting the non-Effect world

An Effect program is a pure description until something runs it. The edges — where
Promises, callbacks, Node APIs, and HTTP handlers meet your effects — are where
most bugs live. Keep the boundary thin and explicit: lift *into* Effect at the
bottom, run *out of* Effect only at the top.

## Promise / callback → Effect

Never call a Promise-returning API bare inside `Effect.gen`. Lift it, and map its
rejection into a typed error:

```ts
// Unknown rejection becomes a defect unless you map it:
yield* Effect.tryPromise({
  try: () => sdk.fetchUser(id),
  catch: (e) => new FetchUserError({ cause: e }),   // → typed failure channel
})

// Promise you trust never to reject meaningfully:
yield* Effect.promise(() => sdk.warmup())

// Callback / event-based API:
yield* Effect.async<Data, ReadError>((resume) => {
  stream.on("data", (d) => resume(Effect.succeed(d)))
  stream.on("error", (e) => resume(Effect.fail(new ReadError({ cause: e }))))
})
```

A useful refinement: a helper that tries a Promise, and **maps only recognized
errors to domain failures, letting everything else become a defect**. This keeps
expected failures typed while genuine bugs stay loud:

```ts
const refineRejection = <A, E>(
  run: () => Promise<A>,
  refine: (cause: unknown) => E | undefined,
) =>
  Effect.tryPromise(run).pipe(
    Effect.catch((err) => {
      const refined = refine(err)
      return refined !== undefined ? Effect.fail(refined) : Effect.die(err)
    }),
  )
```

## Effect → outside world: run only at the top

Effects execute when a runtime runs them. Build one `ManagedRuntime` from your
root layer and run effects through it *only at process entry points* — never deep
inside business logic.

```ts
const runtime = ManagedRuntime.make(AppLayer, { memoMap })

// CLI command / HTTP handler / server start — the imperative edge:
async function handler(args) {
  return runtime.runPromise(program(args))   // back to Promise-land here, and only here
}
```

- `runPromise` / `runPromiseExit` — async edge (most common).
- `runSync` — only for effects you know are synchronous.
- `runFork` — start a long-running effect and get a fiber back.

Everything between the entry point and the leaves stays as `Effect` values. If
you find `runPromise` in the middle of a module, that's a smell — the boundary has
leaked inward.

## Wrapping external resources

Anything with a lifecycle — a connection, a process, an `AbortController`, an
event listener — should be acquired and released as a unit so cleanup is
guaranteed even on failure or interruption:

```ts
// acquire + release (release also runs on interruption):
const controller = yield* Effect.acquireRelease(
  Effect.sync(() => new AbortController()),
  (c) => Effect.sync(() => c.abort()),
)

// acquire + use + release, with the Exit so you can clean up conditionally:
yield* Effect.acquireUseRelease(
  openTransport(),
  (t) => connect(t),
  (t, exit) => Exit.isFailure(exit) ? close(t) : Effect.void,
)

// Bridge a Node EventEmitter into a stream, unsubscribing on scope exit:
const events = Stream.callback<Event>((queue) =>
  Effect.acquireRelease(
    Effect.sync(() => emitter.on("event", (e) => Queue.offerUnsafe(queue, e))),
    () => Effect.sync(() => emitter.removeAllListeners("event")),
  ),
)
```

This is the same `Scope` machinery as service resource lifecycle — see
[services-layers-runtime.md](services-layers-runtime.md) and
[service-graph.md](service-graph.md) §5.

## Carrying context across the boundary

When you hand an effect-derived callback to non-Effect code (an SDK callback, an
event handler), the fiber context doesn't travel with it. Two complementary
fixes:

1. **Capture and restore** — read the ambient values you need *before* crossing
   out, and re-provide them when you cross back in:
   ```ts
   const tenant = yield* TenantRef                       // capture inside the fiber
   sdk.onEvent(() => runtime.runFork(handle().pipe(
     Effect.provideService(TenantRef, tenant),           // restore on re-entry
   )))
   ```
2. **`AsyncLocalStorage` as a fallback** — for imperative code that can't accept
   parameters at all, mirror the context into an ALS store so deep callees can
   read it without threading. Use this only as a secondary channel; the Effect
   `Context.Reference` (see services-layers-runtime) is the primary one.
