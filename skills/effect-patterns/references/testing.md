# Testing Effect code

The thing that makes Effect testable is the same thing that makes it modular:
dependencies are services provided by layers, so a test just provides *different*
layers. No mocking framework, no monkey-patching — you swap implementations at the
layer boundary and run the effect.

## A per-file test harness

Build one small helper that takes the layer under test and returns a `test`
function which provides that layer and runs the effect. A typical shape exposes a
couple of environments:

```ts
// test/lib/effect.ts (sketch)
const testEnv = Layer.mergeAll(TestClock.layer(), TestConsole.layer)   // deterministic
const liveEnv = TestConsole.layer                                       // real clock/fs

export const testEffect = <R, E>(layer: Layer.Layer<R, E>) => ({
  effect: (name, body) => test(name, () =>
    body().pipe(Effect.scoped, Effect.provide(Layer.provideMerge(layer, testEnv)), Effect.runPromise)),
  live:   (name, body) => test(name, () =>
    body().pipe(Effect.scoped, Effect.provide(Layer.provideMerge(layer, liveEnv)), Effect.runPromise)),
})
```

Then each file does `const it = testEffect(myLayer)` and writes
`it.effect("...", () => Effect.gen(...))`. The split matters:

- **`effect`** — `TestClock` + test console: deterministic, instant, no real I/O.
  Use for logic, scheduling, retries.
- **`live`** — real clock and filesystem/process: use for code that genuinely
  touches the OS (running a subprocess, real file reads).

The body is just an `Effect`, run via `Effect.runPromise` inside the harness.
`Effect.scoped` ensures per-test resources are finalized.

## Provide fakes by swapping layers

Two ways to substitute a dependency, depending on how much of it the test
exercises:

```ts
// Stub: only the methods the test calls; un-stubbed methods fail loudly if hit.
const fakeAuth = Layer.mock(Auth.Service)({
  all: () => Effect.succeed({}),
})

// Full fake: a complete Interface implementation as a constant layer.
const fakeProvider = Layer.succeed(
  Provider.Service,
  Provider.Service.of({
    list: () => Effect.succeed({ [row.id]: row }),
    get:  (id) => id === row.id ? Effect.succeed(row) : Effect.die(`unknown: ${id}`),
  }),
)
```

`Layer.mock` is ideal: a method you *didn't* expect to be called dies loudly, so
tests stay honest about what the code under test actually touches. Compose the
real layers you want to exercise with fake layers for everything at the I/O edge
(network, payment SDK, clock), then pass the merged layer to `testEffect`.

As the dependency list grows, a small **layer-composition helper** (a declarative
way to assemble a graph of service layers and override individual ones per test)
keeps test setup from becoming a wall of `Layer.provide` calls. That helper is a
local convenience, not an Effect built-in — but the *idea* (declarative graph +
per-test overrides) transfers.

## First-class stateful fakes: the dual-tag pattern

When a fake needs *control and inspection* methods (record sent messages, inject
the next failure), promote it to a first-class test service: a `TestInterface
extends Interface` with the extra methods, a separate `TestService` tag, and one
`testLayer` where **the same object backs both tags**:

```ts
export interface TestInterface extends Interface {
  readonly sentMessages: () => Effect.Effect<ReadonlyArray<Message>>
  readonly failNextSend: (error: SendError) => Effect.Effect<void>
}
export class TestService extends Context.Service<TestService, TestInterface>()(
  "@app/Notifier/Test",
) {}

export const testLayer = Layer.effectContext(
  Effect.gen(function* () {
    const sent = yield* Ref.make<ReadonlyArray<Message>>([])
    const service = TestService.of({
      send: (msg) => Ref.update(sent, (m) => [...m, msg]),
      sentMessages: () => Ref.get(sent),
      failNextSend: /* ... */,
    })
    return Context.empty().pipe(
      Context.add(Service, service),      // production code sees the real tag
      Context.add(TestService, service),  // tests see the control surface
    )
  }),
)
```

Production code depends only on the real `Service` tag; tests yield
`TestService` to drive and observe. This keeps the fake reusable across test
files instead of re-built inline, and `Layer.effectContext` is the constructor
for exactly this one-acquisition-many-tags case.

## Synchronization instead of sleeps

An arbitrary `Effect.sleep` in a test is a flake waiting to happen. Coordinate
with the concurrency primitives instead ([concurrency.md](concurrency.md)):
`Deferred` for one-shot readiness signals, `Queue` for handing observed events
across fibers, `Latch` for reusable gates, `Ref` for shared observation state —
or an explicit test hook when the production boundary can expose a
deterministic synchronization point. Fork the worker, `Deferred.await` its
readiness, then assert on what lands in the `Queue`.

## Deterministic time

With `TestClock` provided, scheduled effects don't wait in real time — the clock
advances under program control. This is what makes retry/backoff/timeout tests
fast and deterministic instead of flaky `sleep`-based waits. Test a retry policy
by stepping a `Schedule` and asserting on the recorded attempts/delays, with no
wall-clock time elapsed.

One ordering gotcha: **fork the sleeping effect before advancing the clock**
(`Effect.forkScoped`, then `TestClock.adjust`). Advancing first does nothing —
no fiber is waiting on the clock yet.

For config-dependent code, the same swap-the-layer move applies:
`ConfigProvider.layer(ConfigProvider.fromUnknown({...}))` when the test should
exercise Config decoding, or `Layer.succeed(AppConfiguration.Service, config)`
when it just needs values — see [config.md](config.md).

## Asserting on failures

Don't try/catch. Reify the outcome with `Effect.exit` and inspect it:

```ts
const exit = yield* svc.get(id).pipe(Effect.exit)
expect(Exit.isFailure(exit)).toBe(true)
const error = Exit.isFailure(exit) ? Cause.squash(exit.cause) : undefined
expect(error).toBeInstanceOf(NotFoundError)        // or match on _tag / error name
```

`Effect.exit` captures success *and* failure as a value; `Cause.squash` pulls the
underlying error out of the cause for assertion. For the success path you just
assert on the returned value directly. (`Effect.either` is the lighter option when
you only care about the typed error, not the full cause.)

## Fixtures and setup via scope

Per-test setup/teardown is an effect, not a `beforeEach` side effect. Acquire the
fixture in a scoped effect and register cleanup as a finalizer, so it runs no
matter how the test ends:

```ts
const tmpDir = Effect.gen(function* () {
  const dir = yield* makeTempDir
  yield* Effect.addFinalizer(() => removeDir(dir))   // runs on test scope close
  return dir
})
// because the harness wraps the body in Effect.scoped, this cleans up automatically
```

For mutating real global state (an env var, a process-wide singleton), use
`Effect.acquireUseRelease` to save-set-restore around the test body. Reserve a
`afterEach`-style hook for genuinely external registries the scope can't reach.
