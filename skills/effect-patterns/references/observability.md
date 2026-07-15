# Observability: tracing, logging, metrics

Effect has tracing, logging, and metrics built into the effect type, so
observability is mostly *free* if you adopt a few conventions — and it's wired in
once, globally, rather than sprinkled through the code.

## Tracing: spans come from how you define functions

The cheapest win: name your effect-producing functions and they become spans.

```ts
// Traced — creates a span "Billing.charge" whenever this runs:
const charge = Effect.fn("Billing.charge")(function* (order) { /* ... */ })

// Untraced — no span, for internal helpers that would just add noise:
const normalize = Effect.fnUntraced(function* (raw) { /* ... */ })
```

This is the same `Effect.fn("Domain.method")` convention the module contract
already uses (see [module-contract.md](module-contract.md)) — here's *why* it's
worth the discipline: the trace string is the span name. The rule:

- **`Effect.fn(name)`** for operations worth seeing in a trace (service methods,
  externally-visible work).
- **`Effect.fnUntraced`** for hot-path internal helpers — tracing every tiny
  function buries the signal.

For ad-hoc spans, wrap an effect and attach attributes at creation:

```ts
work.pipe(Effect.withSpan("Tool.execute", {
  attributes: { "tool.name": id, "session.id": sessionId },   // dot-notation keys
}))
```

Span attributes are fixed at creation — Effect doesn't do mid-span mutation, so
pass all known context up front. If a *shared* third-party tracer needs
per-call context (e.g. injecting a session id into every span an SDK creates),
wrap its `startSpan` in a `Proxy` and set the attribute there. Use OTel semantic
dot-notation (`service.id`, `tool.name`) so attributes are portable to any
backend.

## Logging: structured, with propagating annotations

Use Effect's leveled loggers and pass context as a structured object, never string
interpolation:

```ts
yield* Effect.logInfo("tracking", { hash, cwd, files: count })
yield* Effect.logWarning("skipping invalid config", { path, reason })
```

Attach context that should appear on *every* log within a scope with
`annotateLogs` — the idiomatic way to thread a request/session id through without
passing it to each call:

```ts
program.pipe(Effect.annotateLogs({ sessionId, tenant }))
```

Nested objects flatten to dot-notation in the output and span context is included
automatically, so a single log line carries timestamp, level, message, your
fields, and the active span — correlatable across concurrent work via an ambient
run id. Choosing the level by calling the matching function (`logDebug` …
`logError`) keeps level filtering a runtime concern, not a code-branching one.

## Metrics: available, opt-in, often deferred

Effect provides `Metric` (counters, gauges, histograms), but **a codebase can run
entirely on traces + structured logs and skip metrics until there's a concrete
need** — spans already carry timing, and structured logs are queryable. Don't add
metrics infrastructure speculatively; add a counter/histogram when a specific SLO
or dashboard requires it. (Being honest that metrics are absent is better than
pretending coverage you don't have.)

## Wire it once, as a global layer

The key architectural move: observability is **one layer provided to everything**,
not a dependency each service declares. Build a layer that sets up loggers
(console/file + optional exporter) and the tracer, and merge it under the app:

```ts
export const Observability = Layer.mergeAll(loggerLayer, tracingLayer)

export const AppLayer = Layer.mergeAll(/* all services */).pipe(
  Layer.provideMerge(Observability),   // every effect can now log and trace
)
```

Services never import a logger or tracer — they just call `Effect.log*` and define
functions with `Effect.fn`, and the global layer decides where it all goes. Make
the exporter **environment-driven**: if an OTLP endpoint env var is set, attach
the exporter; otherwise log locally only. That gives opt-in production telemetry
with zero code change — and zero overhead when it's off.
