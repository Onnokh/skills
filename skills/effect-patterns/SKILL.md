---
name: effect-patterns
description: >-
  Effect-TS (the `effect` package) conventions for TypeScript — use for any
  work building, debugging, or reviewing Effect code, whether the user names
  the APIs or only describes the situation. Triggers: defining a service, its
  Layer, and its dependencies; root Layer / ManagedRuntime problems (e.g.
  layers instantiated more than once); Schema.Struct vs Schema.Class, branded
  IDs, or tagged errors; evolving a persisted/wire schema without a risky
  migration; runtime config via Config/ConfigProvider; retry/backoff
  Schedules; caches with TTL; Stream or queue-backed event sources; outgoing
  HTTP clients with typed errors; and "how do Effect codebases usually
  handle…" questions. Not for generic side effects, React useEffect,
  audio/sound effects, Effect scheduler internals, or non-Effect libraries
  like Zod or Drizzle.
compatibility: >-
  Requires Effect v4. The APIs throughout (Schema.TaggedErrorClass,
  Context.Service, Schema.toTaggedUnion, Config.schema, Layer.effectContext)
  do not exist in v3, and v3 idioms (Effect.Service, Schema.TaggedError) are
  not what this skill teaches.
---

# Effect Patterns

Battle-tested conventions for large Effect-TS codebases. The value is
*uniformity at scale*: every module follows the same skeleton, so any file is
navigable the moment you've seen one — dependency wiring stays type-safe, and
validation, errors, and IDs flow through a single `Schema`-based layer.

**Effect v4 only.** None of this works on v3 — the API surface is different
(`Schema.TaggedErrorClass` vs v3's `Schema.TaggedError`, `Context.Service` vs
`Effect.Service`, and so on). When an API question isn't answered here, check
in order: the nearest `AGENTS.md` / project Effect practices doc, the
project-pinned `effect` package source and version in `node_modules`, and
current upstream Effect source — before guessing from training data, which
skews v3.

## How to use this skill

Read the reference file matching the task. Each is self-contained — load only
what the task needs.

| If the task is about… | Read |
| --- | --- |
| The repeated shape of a module: `Interface` + `Service` + `layer` + `defaultLayer` + self-reexport, no namespaces/barrels, the `serviceUse` accessor | [references/module-contract.md](references/module-contract.md) |
| Dependency injection: `Context.Service`, `Layer.effect`/`sync`/`succeed`/`suspend`, composing one root `Layer`, `ManagedRuntime`, per-instance state, ambient context | [references/services-layers-runtime.md](references/services-layers-runtime.md) |
| Modeling data with `Schema`: `Struct` vs `Class` vs `TaggedClass`, `optional`, unions, annotations, decode/encode, deriving types, `DeepMutable` | [references/schema.md](references/schema.md) |
| Branded, prefix-checked IDs and attaching factory methods with `statics` | [references/branded-ids.md](references/branded-ids.md) |
| Typed errors: `TaggedErrorClass`, reason unions, `Defect`, and catching with `catchTag` | [references/errors.md](references/errors.md) |
| Evolving wire/storage formats with parallel versioned schemas instead of migrations | [references/versioning.md](references/versioning.md) |
| Designing the graph *across* many services: flat composition, hubs vs orchestrators, global vs per-scope scoping, resource lifecycle, tiered runtimes | [references/service-graph.md](references/service-graph.md) |
| Concurrency: forking into scopes, `Ref`/`Deferred`/`Queue`/`PubSub`/`Semaphore`, bounded `concurrency`, interruption/cleanup, background daemons | [references/concurrency.md](references/concurrency.md) |
| Interop with the non-Effect world: `tryPromise`/`async`, running effects at the edges, wrapping resources, carrying context across callbacks | [references/interop.md](references/interop.md) |
| Error *recovery*: `Schedule` retries, timeouts, the `catch*` family, failure vs defect, graceful degradation | [references/error-recovery.md](references/error-recovery.md) |
| Testing Effect code: layer-swapping harness, `Layer.mock`/`Layer.succeed` fakes, stateful dual-tag test services, `TestClock`, asserting via `Effect.exit`, scoped fixtures | [references/testing.md](references/testing.md) |
| Runtime configuration: `Config` recipes, `Config.redacted`/`option`/`withDefault`, `ConfigProvider` layers, `layerConfig` helpers, config in tests | [references/config.md](references/config.md) |
| Caching and memoization: `Cache.make`/`makeWith`, exit-aware TTL, concurrent-lookup dedupe, `Effect.cached`, when `RequestResolver` batching is (and isn't) worth it | [references/caching.md](references/caching.md) |
| `Stream`: source/transform/consume choosers, long-lived consumers in layers, exposing streams from services, backpressure and buffers, keyed concurrency | [references/streams.md](references/streams.md) |
| Outgoing HTTP: Effect `HttpClient`, adapter boundary shape, status classification + schema decoding, `retryTransient`/rate limiting, disciplined raw `fetch` | [references/http-clients.md](references/http-clients.md) |
| Observability: `Effect.fn` spans, structured logging + `annotateLogs`, metrics, one global observability layer | [references/observability.md](references/observability.md) |

## The non-negotiables

These rules are what make the rest cohere. The references explain the *why*
behind each:

1. **One module skeleton, always the same order** — `Interface`, `Service`,
   `use`, `layer`, `defaultLayer`, self-reexport. See module-contract.
2. **No `export namespace` and no barrel `index.ts`** — both defeat
   tree-shaking and Node's native TS. Use flat exports + a self-reexport.
3. **`schema.ts` per domain** — branded IDs, data shapes, and error classes live
   together, separate from the service that uses them.
4. **Errors are `Schema.TaggedErrorClass`** — so they flow through Effect's typed
   error channel and narrow under `catchTag`.
5. **Version by parallel schema files, not migration code** — old formats stay
   readable through their own schemas.
6. **Decide each service's scope explicitly** — global singleton vs per-scope
   (tenant/request) state is the load-bearing call in the graph. See
   service-graph.
7. **Effect ships the hard part — reach for the primitive** — retry is a
   `Schedule`, a per-key cache with TTL/dedupe is `effect/Cache`, config is
   `Config` recipes, a many-valued source is a `Stream`. The hand-rolled
   equivalents (`while`+sleep, `Map`+prune, `process.env`, ad-hoc loops) are
   always missing the hard part: jitter, in-flight dedupe, backpressure,
   interruption.
