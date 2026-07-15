# Typed errors

Errors are `Schema.TaggedErrorClass` definitions. Because they're schemas, they
serialize and validate like any other data; because they're tagged, they flow
through Effect's typed error channel and narrow under `catchTag`. The error type
becomes part of the effect's signature, so the compiler tells you which failures
a caller still has to handle.

## Defining an error

```ts
export class AccountRepoError extends Schema.TaggedErrorClass<AccountRepoError>()(
  "AccountRepoError",
  {
    message: Schema.String,
    cause: Schema.optional(Schema.Defect()),
  },
) {}
```

- The string (`"AccountRepoError"`) is the `_tag` — the discriminator used by
  `catchTag`. Keep it unique across the codebase.
- `Schema.Defect()` carries an arbitrary underlying cause (a caught unknown
  error) without forcing it into a schema. Use it for the `cause` field when
  wrapping a lower-level failure.
- Fields are ordinary schema fields; use the [`optional`](schema.md) helper for
  optional ones.

These appear in effect signatures automatically:

```ts
readonly token: (id: AccountID) => Effect.Effect<string, AccountRepoError>
```

## Errors with a reason union

For an error that fails in several distinct ways, model the *reasons* as a tagged
union and nest it as a field. Getters on the outer error can delegate to the
reason — so callers get one error type with rich, structured detail:

```ts
export const ErrorReason = Schema.Union([
  InvalidRequestReason,     // _tag: "InvalidRequest"
  AuthenticationReason,     // _tag: "Authentication"
  RateLimitReason,          // _tag: "RateLimit", has retryAfterMs
  // ...
]).pipe(Schema.toTaggedUnion("_tag"))

export class ServiceError extends Schema.TaggedErrorClass<ServiceError>()("ServiceError", {
  module: Schema.String,
  method: Schema.String,
  reason: ErrorReason,
}) {
  override readonly cause = this.reason
  get retryable() { return this.reason.retryable }
  get retryAfterMs() {
    return "retryAfterMs" in this.reason ? this.reason.retryAfterMs : undefined
  }
  override get message() {
    return `${this.module}.${this.method}: ${this.reason.message}`
  }
}
```

Each reason is a `Schema.Class`/`TaggedClass` carrying its own fields and its own
`get retryable()`. The outer error stays a single type in signatures while
exposing the variant detail through getters.

## Catching

Tagged errors narrow by `_tag`, so you handle exactly the cases you mean to and
the compiler tracks the rest:

```ts
program.pipe(
  Effect.catchTag("AccountRepoError", (err) =>
    Effect.logError(err.message).pipe(Effect.as(fallback)),
  ),
  Effect.catchTags({
    InvalidRequest: (e) => /* ... */,
    RateLimit: (e) => Effect.sleep(e.retryAfterMs ?? 1000),
  }),
)
```

After a `catchTag`, that tag drops out of the effect's error type — the signature
reflects what's left to handle.

## When to use a lighter helper

For trivial errors that only carry a couple of fields and never need schema
serialization, a thin `NamedError.create("Tag", { ...fields })` factory is an
acceptable shorthand. Reach for full `TaggedErrorClass` when the error needs
proper schema fields, a `Defect` cause, a reason union, or computed getters —
i.e. anything beyond a name and a message.

## Why schemas for errors

- **Typed channel** — failures are in the effect type, not stringly-typed throws.
- **Serializable** — a `TaggedErrorClass` crosses a process/wire boundary and
  decodes back into the same class.
- **Narrowable** — `catchTag`/`catchTags` give exhaustive, compiler-checked
  handling per failure mode.
