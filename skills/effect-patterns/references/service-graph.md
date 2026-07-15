# Shaping the service graph

Once you have more than a handful of services, the *graph they form* — who
depends on whom, what's a singleton, what holds resources — matters more than any
single module. These are heuristics for keeping that graph healthy as it grows
into dozens of services. They're independent of any particular domain; the
examples (`Config`, `EventBus`, `BillingService`, …) are illustrative stand-ins
for whatever your services actually are.

## 1. Compose flat; let the type system resolve the order

Don't hand-wire the dependency order. List every layer in one `Layer.mergeAll`,
then `provideMerge` the shared infrastructure underneath. Effect resolves the DAG
by type — a service that yields `Config.Service` gets `Config` no matter where it
sits in the merge.

```ts
export const AppLayer = Layer.mergeAll(
  Config.defaultLayer,
  EventBus.defaultLayer,
  BillingService.defaultLayer,
  OrderService.defaultLayer,
  // ...every service, in any order
).pipe(
  Layer.provideMerge(Database.defaultLayer),   // shared infra for all of the above
  Layer.provideMerge(HttpClient.defaultLayer),
)
```

The payoff: this file stays tiny even with 50 services, and **adding a service is
a one-line change** plus declaring its `yield*` dependencies. The dependency
*chain* can be 10+ deep and you never order it by hand. If this composition file
starts needing comments to explain ordering, something has broken the convention
— flat composition should never require manual sequencing.

## 2. Make cross-cutting concerns leaf services, not ambient globals

Anything that most modules need — configuration, an event bus, a clock, a logger,
a feature-flag reader — should be a **dependency-free service at the base of the
graph**, injected like any other, *not* a module-level singleton imported
directly.

```ts
// Good: depend on it explicitly, swap it in tests
const bus = yield* EventBus.Service
yield* bus.publish(event)

// Avoid: import { globalBus } from "./bus"  — invisible coupling, hard to fake
```

A healthy graph has a few **leaves** (zero service dependencies: `Env`,
`Database`, `EventBus`, `Clock`) that many other services depend *on*. It's
normal and good for one service to be both a leaf and the most-depended-upon node
— a primitive everything builds on. Because it's injected, every consumer can
substitute a fake in tests without touching globals.

## 3. Distinguish hubs from orchestrators — they fail differently

Two different "central" roles, with opposite shapes:

- **Hubs** = high fan-*in* (many services depend on them): `Config`, `EventBus`.
  Keep these **small, stable, and slow-changing** — a breaking change ripples to
  every dependent. Their interface is a contract for the whole codebase.
- **Orchestrators** = high fan-*out* (they depend on many services): a "handle a
  request end-to-end" service that pulls in 15 others. Keep these **thin** —
  coordination only, with the real work delegated to the services they call. A
  fat orchestrator is where logic goes to become untestable.

When reviewing, a service that is *both* high fan-in and high fan-out is a smell:
it's a bottleneck that's hard to change and hard to test. Split it.

## 4. The service is your scoping boundary

For each service, make one explicit decision: **is its state shared process-wide,
or must each scope (tenant / request / workspace / connection) get its own
copy?** This is the single most important call in the graph.

- **Global singleton** → a plain layer in the root. Correct for stateless logic
  and genuinely shared resources: a connection pool, config loaded once, a
  stateless calculator.
- **Per-scope state** → a per-scope cache (e.g. Effect's `ScopedCache`) keyed by
  the scope identity, lazily initialized on first use. Correct whenever two
  concurrent scopes must *not* see each other's data: a per-tenant client, a
  per-request in-flight queue, a per-connection session.

The test to apply: *if two active scopes shared one copy of this, would that be a
bug?* If yes, it's per-scope. Getting this wrong is the classic multi-tenant leak
(tenant A sees tenant B's cached data) or, the opposite, needless re-initialization
of something that could be shared.

## 5. Tie resources to a `Scope` so cleanup is structural

Any service that holds a real resource — a spawned process, socket, file handle,
external connection — should acquire it inside a `Scope` (via `acquireRelease`
or `Scope.Scope`), not manage open/close by hand.

```ts
const layer = Layer.scoped(
  Service,
  Effect.gen(function* () {
    const conn = yield* Effect.acquireRelease(
      openConnection(),
      (c) => Effect.promise(() => c.close()),   // runs automatically on scope exit
    )
    return { /* methods using conn */ }
  }),
)
```

When the scope ends — process shutdown, request completion, or invalidation of a
per-scope cache entry — Effect runs the finalizer. Cleanup becomes a property of
the lifecycle rather than something a human has to remember at every exit path.
This is what makes "per-scope resource + dispose on scope end" leak-free: the
resource's lifetime *is* the scope's lifetime, by construction.

## 6. Carry scope identity ambiently, not through every signature

When a value is needed deep in the graph but is a property of the *current scope*
(which tenant/request am I serving?), threading it through every function
signature pollutes the whole codebase. Use a `Context.Reference` — a context entry
read inside any effect — and set it once at the boundary:

```ts
export const TenantRef = Context.Reference<TenantId | undefined>(
  "TenantRef", { defaultValue: () => undefined },
)

// at the entry point (HTTP middleware, job handler):
effect.pipe(Effect.provideService(TenantRef, tenantId))

// anywhere deep inside, with no parameter threading:
const tenant = yield* TenantRef
```

Reserve this for genuinely ambient, scope-bound values. The static dependency
graph still belongs in services; `Reference` is only for "where am I running",
not "what do I depend on".

## 7. Decompose a big domain into sibling services, not a god-service

When one domain grows large, resist a single 2,000-line service. Split it into
focused sibling services that each own one concern (e.g. for a "session" domain:
lifecycle, the processing loop, summarization, revert, status) and depend on each
other through the normal contract. The domain stays large in total but every file
stays small and independently testable. A large *line count* in a domain is fine;
a large *single service* rarely is.

## 8. Watch what actually inflates a service

File size and interface size are nearly uncorrelated. Two different things make a
service big, and they call for different responses:

- **Many schema definitions / integration glue** — a service wrapping several
  external SDKs or holding 20 `Schema` shapes. Often fine, but consider moving
  the schemas to a co-located `schema.ts` (see [schema.md](schema.md)) so the
  service file is logic, not type declarations.
- **Many dependencies (fan-out)** — that's an orchestrator; see §3, keep it thin.

A service with few methods but huge line count is usually schema/SDK weight; a
service with many dependencies is usually doing too much coordination. Diagnose
which before "simplifying".

## 9. Optional: tier the runtime for fast common paths

If some entry points need only a few services and must start instantly (a CLI
command) while others need the full graph (a long-running server), split the
composition into a small **bootstrap layer** (the few cheap, immutable services)
and the **full app layer**. The cheap path pays only for what it uses; the full
path builds on top. Lazily initialize the expensive per-scope state so even the
full graph doesn't construct everything up front. This is an optimization — reach
for it only when startup cost or memory of unused services is a measured problem,
not by default.
