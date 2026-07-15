# Branded IDs

Domain identifiers are branded strings with a checked prefix, plus factory
methods attached directly to the schema. The brand stops a `SessionID` from being
passed where a `MessageID` is expected; the prefix check (`"msg"`, `"ses"`, …)
catches malformed or mismatched IDs at decode time; the attached factory keeps
generation right next to the type it produces.

## The shape

```ts
export const MessageID = Schema.String.check(Schema.isStartsWith("msg")).pipe(
  Schema.brand("MessageID"),
  statics((s) => ({
    ascending: (id?: string) => s.make(Identifier.ascending("message", id)),
  })),
)
export type MessageID = Schema.Schema.Type<typeof MessageID>
```

Three things stacked with `.pipe`:
1. `Schema.String.check(Schema.isStartsWith("msg"))` — must be a string starting
   with the prefix.
2. `Schema.brand("MessageID")` — nominal type so it can't be confused with other
   string IDs.
3. `statics(...)` — bolts factory methods onto the schema object itself.

A plain branded ID without a prefix check is fine when there's no meaningful
prefix:

```ts
export const AccountID = Schema.String.pipe(Schema.brand("AccountID"))
export type AccountID = Schema.Schema.Type<typeof AccountID>
```

## The `statics` helper

`statics` is a tiny utility that merges extra properties onto a schema, so a
schema can carry its own constructors (`MessageID.ascending()`) rather than
exposing a separate free function:

```ts
export const statics =
  <S extends object, M extends Record<string, unknown>>(methods: (schema: S) => M) =>
  (schema: S): S & M =>
    Object.assign(schema, methods(schema))
```

The callback receives the schema, so factories can call `schema.make(...)` to
produce a value already typed with the brand:

```ts
export const ID = Schema.String.check(Schema.isStartsWith("per")).pipe(
  Schema.brand("Permission.ID"),
  statics((schema) => ({
    create: (id?: string) => schema.make(id ?? "per_" + ascending()),
  })),
)
```

## Generating the value

Back the factories with one identifier generator, keyed by a central prefix
table. IDs are time-ordered (a timestamp component + random bytes,
base62-encoded) so they sort chronologically — useful for cursoring and stable
ordering:

```ts
const prefixes = {
  session: "ses", message: "msg", part: "prt", permission: "per",
  event: "evt", job: "job", workspace: "wrk",
} as const

export function ascending(prefix: keyof typeof prefixes, given?: string) {
  return generateID(prefix, "ascending", given)   // timestamp-prefixed, monotonic, base62
}
```

`MessageID.ascending()` → `Identifier.ascending("message")` → a value like
`msg_0xX...`. Centralizing the prefix table keeps prefixes unique and prevents two
entities from colliding on one prefix.

## Why this is worth it

- **Type safety across boundaries** — function signatures that take a `SessionID`
  reject a raw string or a `MessageID`.
- **Validation at the edge** — decoding untrusted input through the ID schema
  rejects wrong-prefixed values immediately, with a clear message.
- **Discoverability** — `SomeID.ascending()` / `SomeID.create()` lives on the type,
  so the way to make one is obvious from the type itself.
