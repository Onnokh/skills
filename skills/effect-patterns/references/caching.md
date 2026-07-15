# Caching, memoization, and request dedupe

The rule that pays for this file: **don't hand-roll a `Map` + timestamp + prune
loop**. `effect/Cache` already does keyed memoization, TTL, capacity bounds,
and — the part people always reimplement badly — deduplication of concurrent
lookups for the same key. Reach for it whenever you're memoizing expensive
lookups or caching per-key results.

## The core API

- `Cache.make({ capacity, lookup, timeToLive })` — per-key lookups, one fixed
  TTL for all entries.
- `Cache.makeWith(lookup, { capacity, timeToLive(exit, key) })` — TTL computed
  per entry from the lookup's `Exit`. This is the tool for "cache successes,
  not failures".
- Concurrent `Cache.get` calls for a missing key **share one pending lookup** —
  dedupe is built in; never add your own in-flight tracking on top.
- `capacity` is required and bounds the cache — no manual prune/evict loops.
- `Cache.invalidate(cache, key)` / `Cache.refresh(cache, key)` for explicit
  staleness; `Cache.has` checks without triggering a lookup.
- Single value, no key: `Effect.cached(effect)` / `Effect.cachedWithTTL(effect, ttl)`
  instead of a one-key Cache.
- Cached resources that need cleanup (connections, clients): `ScopedCache`.

## Exit-aware TTL: cache successes, skip degraded results

Return a zero TTL from `timeToLive` to avoid caching transient failures without
failing the caller. A short *negative-cache* TTL is legitimate for stable
failures like not-found:

```ts
const cache = yield* Cache.makeWith(
  (ref: string) => resolveUncached(ref),   // never-failing, returns { where, cacheable }
  {
    capacity: 300,
    timeToLive: (exit) =>
      Exit.isSuccess(exit) && exit.value.cacheable ? "10 minutes" : Duration.zero,
  },
)
```

This replaces the hand-rolled `Map<string, { value, expiresAtMs }>` and
upgrades it: a burst of rows hitting the same key shares a single provider call.

## Build the cache in the owning layer, not per call

Cache construction is effectful. A cache built inside a request handler caches
nothing — hoist it to the layer that owns the lifetime and share the handle
(see [service-graph.md](service-graph.md) for scoping decisions).

Same principle one level down: **a cache can't fix a lookup that pays a scoped
acquisition per call**. If the lookup constructs an SDK client or
authenticates, acquire the client once in the layer so misses are one plain
API call:

```ts
// Bad: every cache miss acquires a fresh client
const lookup = (id: string) =>
  getRecord(id).pipe(Effect.provide(apiClientLayer(options)))

// Good: client built once for the layer's lifetime
const context = yield* Layer.build(apiClientLayer(options))  // needs Scope
const lookup = (id: string) => Context.get(context, ApiClient).getRecord(id)
```

## Request batching (`Effect.request` + `RequestResolver`)

Batching exists for backends with a **real batch endpoint** — the resolver
receives an array of pending requests and collapses them into one wire call
(SQL `IN (...)`, DataLoader-style endpoints, batch GET). If the backend only
has per-item endpoints, a batched resolver still loops one call per entry and
buys nothing over `Effect.forEach(items, f, { concurrency })`.

Selection guide:

| Situation | Tool |
| --- | --- |
| Same key requested repeatedly over time | `Cache` |
| Same key requested concurrently in one burst | `Cache` (shared pending lookup) |
| Many distinct keys, backend has a batch endpoint | `Effect.request` + `RequestResolver` |
| Many distinct keys, per-item endpoint only | `Effect.forEach(..., { concurrency: n })`, optionally through a `Cache` |

`RequestResolver.batchN(resolver, n)` bounds batch size;
`RequestResolver.makeGrouped` groups requests resolving through different
targets.
