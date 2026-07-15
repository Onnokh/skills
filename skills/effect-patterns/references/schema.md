# Modeling data with Schema

`Schema` (from the `effect` package) is the single layer for validation, type
derivation, and serialization. Define a schema once; derive the TypeScript type
from it; decode untrusted input through it. Don't hand-write a type and a
validator separately — they drift.

## Struct vs Class vs TaggedClass

| Construct | Use for | Why |
| --- | --- | --- |
| `Schema.Struct({...})` | plain data records | lightest weight; just shape + validation |
| `Schema.Class<T>("Id")({...})` | data that needs **computed getters or helper methods** | a real class, so you can add `get`/methods alongside validation |
| `Schema.TaggedClass<T>()("Tag", {...})` | a **non-error** variant in a discriminated union | injects the `_tag` literal for you |
| `Schema.TaggedErrorClass<T>()("Tag", {...})` | **errors** | see [errors.md](errors.md) — flows through Effect's error channel |

```ts
// Struct — pure record
export const Request = Schema.Struct({
  id: ID,
  action: Schema.String,
  resources: Schema.Array(Schema.String),
  metadata: Schema.Record(Schema.String, Schema.Unknown).pipe(optional),
}).annotate({ identifier: "Request" })

// Class — needs a computed property
export class AuthenticationReason extends Schema.Class<AuthenticationReason>("Error.Authentication")({
  _tag: Schema.tag("Authentication"),
  message: Schema.String,
  kind: Schema.Literals(["missing", "invalid", "expired", "unknown"]),
}) {
  get retryable() { return false }
}

// TaggedClass — union variant
export class PollSuccess extends Schema.TaggedClass<PollSuccess>()("PollSuccess", {
  email: Schema.String,
}) {}
export class PollPending extends Schema.TaggedClass<PollPending>()("PollPending", {}) {}
export const PollResult = Schema.Union([PollSuccess, PollPending])
```

**Rule of thumb:** reach for `Struct` first. Upgrade to `Class` only when you
need behavior (a getter, a method) attached to the data.

## Deriving the type

Never hand-write the type next to the schema. Derive it, so the two can't drift:

```ts
export const MessageID = /* ...schema... */
export type MessageID = Schema.Schema.Type<typeof MessageID>

// For object schemas, the `interface extends` form gives cleaner editor tooltips:
export const Info = Schema.Struct({ /* ... */ }).annotate({ identifier: "Info" })
export interface Info extends Schema.Schema.Type<typeof Info> {}
```

## DeepMutable — when you need mutable derived types

Schema decodes to **`readonly`** properties by default, which is what you want
most of the time. When data is built up progressively (config assembled from
several sources, items collected into an array and then mutated), wrap the
derived type:

```ts
export type Config = DeepMutable<Schema.Schema.Type<typeof ConfigSchema>>
```

Prefer a *local* `DeepMutable` over the one bundled with libraries — the common
upstream version collapses `unknown` to `{}` and loses readonly tuples. A local
copy that strips `readonly` recursively while preserving `unknown` and tuples
avoids those bugs:

```ts
export type DeepMutable<T> = T extends string | number | boolean | bigint | symbol | Function
  ? T
  : T extends readonly [unknown, ...unknown[]]
    ? { -readonly [K in keyof T]: DeepMutable<T[K]> }
    : T extends readonly (infer U)[]
      ? DeepMutable<U>[]
      : T extends object
        ? { -readonly [K in keyof T]: DeepMutable<T[K]> }
        : T
```

Keep mutability the exception. Most schema-derived types should stay `readonly`.

## The custom `optional` helper

Bare `Schema.optional` allows the key to be present with an `undefined` value.
The idiomatic project helper instead makes the **key absent** on encode (so
serialized output is clean) while still decoding leniently. Define it once and
use it everywhere via `.pipe(optional)`:

```ts
export const optional = <S extends Schema.Top>(schema: S) =>
  Schema.optionalKey(schema).pipe(
    Schema.decodeTo(Schema.optional(Schema.toType(schema)), {
      decode: SchemaGetter.passthrough({ strict: false }),
      encode: SchemaGetter.transformOptional(Option.filter((v) => v !== undefined)),
    }),
  )

// usage
metadata: Schema.Record(Schema.String, Schema.Unknown).pipe(optional),
parentID: ID.pipe(optional),
```

## Unions, literals, annotations

```ts
// discriminated union by a `type` field
export const Info = Schema.Union([Oauth, Api, WellKnown])
  .annotate({ discriminator: "type", identifier: "Auth" })

// tagged union — discriminate strictly by `_tag`
export const Reason = Schema.Union([ReasonA, ReasonB]).pipe(Schema.toTaggedUnion("_tag"))

// enums
kind: Schema.Literals(["allow", "deny", "ask"]),     // several values
role: Schema.Literal("user"),                         // one value
```

`.annotate({ identifier: "..." })` names the schema in error messages and
codegen — set it on anything that appears in user-facing validation output.
`discriminator` / `toTaggedUnion` make unions narrow predictably.

## `Data.TaggedEnum` for internal control flow

Not every variant needs a schema. For **internal** workflow decisions and state
machines that never cross a wire or get persisted, `Data.TaggedEnum` gives you
constructors, `$is` guards, and exhaustive `$match` without any
decoding machinery:

```ts
type Step = Data.TaggedEnum<{
  Continue: { readonly cursor: number }
  Finished: { readonly count: number }
}>
export const Step = Data.taggedEnum<Step>()

const next = Step.Continue({ cursor: 10 })
const label = Step.$match(next, {
  Continue: ({ cursor }) => `continue at ${cursor}`,
  Finished: ({ count }) => `finished ${count}`,
})
```

The split is principled rather than forced: **`Data` internally, `Schema` at
boundaries**. Don't add a Schema to an internal-only union solely to obtain
match utilities — and conversely, anything decoded, encoded, persisted, or
validated stays a Schema union.

## Reusing fields across related contracts

When contracts are genuinely related (create-input vs stored row vs API
response), reuse fields instead of copy-pasting shapes that will drift:

```ts
export const CreateUserInput = Schema.Struct({
  name: User.fields.name,
  email: User.fields.email,
})

export const StoredUser = User.pipe(
  Schema.fieldsAssign({ createdAt: Schema.DateTimeUtcFromString }),
)
```

`.fields`, `Schema.fieldsAssign(...)`, and `.mapFields(...)` cover
pick/extend/transform; `Schema.encodeKeys(...)` handles the case where decoded
TypeScript names differ from wire/storage keys and naming is the *only*
difference. Two guardrails: keep an explicit mapping when real behavior
(joins, validation, domain translation) is involved, and use field reuse to
build small related contracts — not one oversized inheritance-by-schema
object.

## Decode and encode

Pick the decode variant by how you want failures surfaced:

| Function | Returns | Use when |
| --- | --- | --- |
| `Schema.decodeUnknownEffect(s)` | `Effect<T, ParseError>` | the **default at untrusted boundaries** — failure stays in the typed error channel |
| `Schema.decodeUnknownExit(s)(data, { errors: "all" })` | `Exit<T, ParseError>` | you want the **full error detail** to reformat for the user (config parsing) |
| `Schema.decodeUnknownOption(s)` | `Option<T>` | invalid input should be **silently dropped**, not reported |
| `Schema.decodeUnknownSync(s)` | `T` (throws) | scripts, tests, startup paths — trusted/inline data where a throw is acceptable |

Construction follows the same trust line: `schema.make(...)` (throws) for
trusted in-process construction, `schema.makeEffect(...)` when construction
failure should stay in the error channel — and never a cast to skip validation.

Surfacing decode errors cleanly — squash the cause, format the issues, wrap in a
domain error:

```ts
const decoded = Schema.decodeUnknownExit(schema)(data, { errors: "all", propertyOrder: "original" })
if (Exit.isSuccess(decoded)) return decoded.value as DeepMutable<S["Type"]>

const error = Cause.squash(decoded.cause)
throw new InvalidError({
  path: source,
  issues: Schema.isSchemaError(error)
    ? SchemaIssue.makeFormatterStandardSchemaV1()(error.issue).issues.map((i) => ({
        message: i.message,
        path: i.path?.map(String) ?? [],
      }))
    : [{ message: String(error), path: [] }],
})
```

## Co-location

Keep schemas and their derived types in a `schema.ts` per domain, beside the IDs
([branded-ids.md](branded-ids.md)) and error classes ([errors.md](errors.md))
they belong with — not inside the service file. The service imports them. This
keeps the data model independently readable and reusable.
