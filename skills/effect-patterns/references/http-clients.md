# Outgoing HTTP

Prefer Effect's HTTP client modules (`HttpClient`, `HttpClientRequest`,
`HttpClientResponse`, `HttpClientError`, all from `effect/http`) for outgoing
HTTP in application and provider code — their typed errors, layers, and client
transforms compose with everything else in this skill. Raw `fetch` remains a
deliberate choice for platform transports, browser/edge constraints, or
libraries that intentionally avoid the HTTP modules, which are still marked
`@stability unstable`.

## The adapter owns the whole boundary

An HTTP adapter method is a named effect (`Effect.fn("Provider.operation")`)
that owns the full round trip, in order:

1. construct request
2. attach auth and headers
3. execute
4. classify status
5. decode the response body with Schema
6. map transport/status/decode failures to typed domain errors
   ([errors.md](errors.md))
7. apply retry/rate-limit policy where the operation is idempotent

Business services call the adapter and see only typed domain errors — raw
provider/network effects stay out of business logic and out of database
transactions.

## The useful API surface

- `HttpClient.get/post/put/patch/del/execute` — service accessors.
- `FetchHttpClient.layer` — the client backed by `fetch`. `FetchHttpClient.Fetch`
  is a `Context.Reference`; `Effect.provideService(FetchHttpClient.Fetch, myFetch)`
  swaps the fetch function (tests, instrumented transports).
- `HttpClient.mapRequest(...)` / `mapRequestEffect(...)` — configured client
  transforms (base URL, auth applied once for the whole client).
- `HttpClientRequest.prependUrl(...)`, `.bearerToken(...)`, `.acceptJson`.
- `HttpClientRequest.bodyJson(...)` / `schemaBodyJson(...)` — JSON body
  encoding, plain or schema-backed.
- `HttpClient.filterStatusOk` / `HttpClientResponse.filterStatusOk` — classify
  status *before* decoding when non-2xx is a failure.
- `HttpClientResponse.schemaBodyJson(...)` (body only), `schemaJson(...)`
  (status/headers/body), `schemaNoBody(...)` (status/headers).

## Retry and rate limits

Three tiers, from generic to domain-aware:

- `HttpClient.retryTransient(...)` — the standard sweep for transport errors,
  timeouts, and 408/429/500/502/503/504, honouring `Retry-After`. Always pass
  `times` or a `schedule`: with neither it retries without limit. Order
  matters: an `HttpClient.transform` piped *before* it runs once per attempt,
  one piped after wraps all attempts.
- `HttpClient.withRateLimiter(...)` — proactive pacing that learns from
  rate-limit / `Retry-After` headers. Requires a `limiter` plus initial
  window/limit/key options; adds `RateLimiterError` to the error channel and
  retries 429 by default. `RateLimiter` lives in `effect/persistence`, and
  `RateLimiter.layer` needs a store (`RateLimiter.layerStoreMemory`, or the
  Redis store for limits shared across processes). Called directly,
  `consume({ onExceeded: "delay" })` returns the delay and does not sleep; use
  `RateLimiter.sleep` or `makeWithRateLimiter` to wait.
- Operation-level `Effect.retry(...)` — when retry depends on domain-specific
  typed errors or idempotency rules, e.g. a schedule that honors a provider's
  `retryAfterMs` field. See the rate-limit-aware schedule in
  [error-recovery.md](error-recovery.md).

Retry only operations with proven idempotency, at the narrowest boundary that
can respond truthfully.

## Raw `fetch`, done with discipline

When raw fetch is unavoidable, keep it inside an adapter service and apply the
same boundary shape:

```ts
const request = Effect.fn("Provider.request")(function* (input: RequestInput) {
  const response = yield* Effect.tryPromise({
    try: (signal) => fetch(input.url, { signal, headers: input.headers }),
    catch: (cause) => new ProviderError({ operation: "Provider.request", cause }),
  })

  if (!response.ok) {
    return yield* Effect.fail(new ProviderRejected({
      operation: "Provider.request",
      status: response.status,
    }))
  }

  const json = yield* Effect.tryPromise({
    try: () => response.json(),
    catch: (cause) => new ProviderError({ operation: "Provider.decodeJson", cause }),
  })

  return yield* Schema.decodeUnknownEffect(ResponseSchema)(json).pipe(
    Effect.mapError((cause) =>
      new ProviderError({ operation: "Provider.decodeResponse", cause }),
    ),
  )
})
```

Note the details that matter: the `AbortSignal` from `Effect.tryPromise` is
wired into `fetch` (so interruption cancels the request), status is classified
before decoding, and the unknown body goes through Schema. Preserve provider
evidence needed for diagnosis, but redact secrets.
