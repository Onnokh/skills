# Services, layers, and the runtime

How dependency injection, layer composition, the application runtime, and
per-instance state fit together. The mental model: **services declare what they
need by type, layers supply it, one root layer composes everything, and a
`ManagedRuntime` turns that into something you can actually run.**

## Services as the DI primitive

A service is a `Context.Service` subclass whose type parameter is its public
`Interface`. The tag string is its runtime identity.

```ts
export interface Interface {
  readonly get: (key: string[]) => Effect.Effect<unknown, NotFoundError>
}
export class Service extends Context.Service<Service, Interface>()("@app/Storage") {}
```

Dependencies are pulled inside a layer by *yielding the other service's tag* —
you ask for it by type, Effect supplies it from context:

```ts
export const layer = Layer.effect(
  Service,
  Effect.gen(function* () {
    const fs = yield* FSUtil.Service       // FSUtil.Service must be provided somewhere upstream
    return { get: Effect.fn("Storage.get")(function* (key) { /* uses fs */ }) }
  }),
)
```

## Choosing a Layer constructor

| Constructor | Use when |
| --- | --- |
| `Layer.effect(Service, eff)` | **default** — building the service requires effects (reading config, opening resources, yielding other services) |
| `Layer.sync(Service)(() => impl)` | construction is pure but stateful (e.g. a `Set` of sockets) and needs no effectful deps |
| `Layer.succeed(Service, value)` | the service is a plain constant |
| `Layer.suspend(() => layer.pipe(...))` | defer composing a heavy dependency graph until first use |
| `Layer.effectContext(eff)` | one acquisition intentionally supplies **multiple tags** — one client backing several services, or a stateful fake backing both `Service` and `TestService` (see [testing.md](testing.md)) |
| `Layer.unwrap(eff)` | config or runtime discovery **chooses/builds** the layer |
| `Layer.effectDiscard(eff)` | the layer provides no service — it exists to start scoped background work (a stream consumer, a daemon; see [streams.md](streams.md)) |

One lifecycle rule for all of them: a layer that starts a stream, listener,
worker, or forever loop must **fork that work into the layer scope**
(`Effect.forkScoped`, `FiberSet`, `FiberMap`) — layer acquisition itself must
complete. Don't run forever work inline during acquisition, and don't expose a
public `start` method unless the domain genuinely needs manual lifecycle
control.

Provide a layer's own dependencies with `Layer.provide`, and expose a
`defaultLayer` that is "the layer with everything it needs already wired":

```ts
export const defaultLayer = layer.pipe(
  Layer.provide(FSUtil.defaultLayer),
  Layer.provide(Config.defaultLayer),
)
```

## The single root layer

Compose every service's layer into one application layer with `Layer.mergeAll`,
then provide cross-cutting infrastructure underneath with `Layer.provideMerge`:

```ts
export const AppLayer = Layer.mergeAll(
  Storage.defaultLayer,
  Auth.defaultLayer,
  Config.defaultLayer,
  Provider.defaultLayer,
  // ...every service
).pipe(
  Layer.provideMerge(Ripgrep.defaultLayer),   // shared infra available to all of the above
  Layer.provideMerge(InstanceLayer.layer),
)
```

`mergeAll` puts the services side by side; `provideMerge` supplies a dependency
to the whole merged set *and* keeps it in the resulting context. Build the graph
once, here.

Two disciplines keep the root layer honest. Use `Layer.provide` when a
dependency is an implementation detail that downstream consumers shouldn't see,
and reserve `provideMerge` for dependencies that genuinely belong in the shared
context (infrastructure like the observability layer) — reaching for
`provideMerge`/`mergeAll` just to make the types compile hides the real graph.
And keep important authority (credentials, persistence, transports) *visible*
in the composition rather than buried behind broad invisible provisioning; a
flat, topologically-sorted layer value with named subgraphs stays auditable.

## ManagedRuntime

A `Layer` is a recipe; a `ManagedRuntime` is the running result you execute
effects against. Build it once and reuse it — pass a shared `memoMap` so layers
that appear in multiple places are instantiated only once.

```ts
export const Runtime = ManagedRuntime.make(AppLayer, { memoMap })
// later:
await Runtime.runPromise(SomeModule.use.doThing(args))
```

A small `makeRuntime` helper that always threads the same `memoMap` keeps layer
deduplication automatic. Route all service execution through it rather than
constructing ad-hoc runtimes.

## Per-instance state

The runtime is process-global, but a tool that operates on multiple working
directories (or tenants, or projects) at once must not share one copy of a
service's mutable state across them. That's the job of a per-instance state
helper backed by a scoped cache:

```ts
// inside a layer
const state = yield* InstanceState.make((ctx) =>
  Effect.gen(function* () {
    // initialize state for THIS instance (e.g. this directory)
    return { /* ... */ }
  }),
)
// retrieve (lazily initializes, then caches, keyed by the current instance):
const current = yield* InstanceState.get(state)
```

`InstanceState` wraps a `ScopedCache` keyed by the instance identifier
(e.g. directory), initializes lazily on first `get`, caches afterward, and
registers a disposer so cleanup happens when the scope closes.

**Decision rule:** if two open instances should *not* share one copy of the
service's state, it needs `InstanceState`. If the state is genuinely global
(a connection pool, a config loaded once), a plain layer is correct.

## Ambient context across async boundaries

Effect's `Context` carries the static dependency graph. For *ambient,
request-scoped* values — "which instance/workspace am I currently serving?" —
two complementary tools:

- **`Context.Reference`** — a context entry with a default value, read inside
  effects without threading it through every signature:
  ```ts
  export const InstanceRef = Context.Reference<InstanceContext | undefined>(
    "~app/InstanceRef",
    { defaultValue: () => undefined },
  )
  ```
- **`AsyncLocalStorage`** — for ambient context at the boundary with non-Effect
  code (Node callbacks, framework handlers):
  ```ts
  export function create<T>(name: string) {
    const storage = new AsyncLocalStorage<T>()
    return {
      use() {
        const v = storage.getStore()
        if (!v) throw new NotFound(name)
        return v
      },
      provide<R>(value: T, fn: () => R) { return storage.run(value, fn) },
    }
  }
  ```

Use Effect `Context`/services for the dependency graph; reach for `Reference` /
`AsyncLocalStorage` only for ambient scope that would otherwise pollute every
function signature.
