# Versioning schemas

When a wire or storage format evolves, **add a new schema version alongside the
old one** rather than writing migration code that rewrites stored data. Each
version is an independent, self-identifying schema. Old data stays readable
through its own version's schema; new data is written in the new shape from the
adoption boundary forward.

## Layout

Keep superseded versions in a versioned subfolder; the current version lives at
the top level of the domain:

```
schema/
├── session.ts          # current (V2)
├── permission.ts       # current (V2)
└── v1/
    ├── session.ts       # legacy V1 — still imported to read old data
    └── permission.ts
```

Each version annotates itself so its identity is explicit in errors and tooling:

```ts
// schema/v1/session.ts
export const User = Schema.Struct({ /* ...old shape... */ })
  .annotate({ identifier: "SessionV1.User" })

// schema/session.ts
export const Info = Schema.Struct({ /* ...new shape... */ })
  .annotate({ identifier: "SessionV2.Info" })
```

Reference them by version-qualified names at call sites (`SessionV1.User`,
`SessionV2.Info`) so it's always obvious which format a piece of code handles.

## Versioned events

When events are persisted or sent over the wire, stamp the version onto the event
type and resolve it on read, so a consumer knows exactly which schema to decode
with:

```ts
const type = EventV2.versionedType(event.type, event.durable.version)
```

A small bridge layer translates between versions at the runtime boundary —
reading a V1 event and surfacing it in the current shape — keeping version logic
in one place instead of scattering `if (version === 1)` through the codebase.

## Why no migration layer

- **Old data stays valid.** It was written correctly under its version; its
  schema still decodes it. No risky bulk rewrite, no migration that can corrupt
  history.
- **Each version is independently testable.** A V1 schema is just a schema —
  decode a fixture and assert.
- **The boundary is explicit.** New writes use the new schema from a known point;
  reads pick the schema by the data's stamped version. There's no ambiguous
  "half-migrated" state.

## When you *do* need transformation

Reading-as-current sometimes requires mapping old fields to new. Keep that
mapping in the bridge/translation layer as a pure function from the V1 type to the
V2 type — not as an in-place mutation of stored records. The store keeps the
original; transformation happens on read.

## Trade-off to be aware of

Parallel versions mean the old schemas linger in the codebase. That's the
intended cost: a little dead-feeling code in exchange for never running a
destructive migration. Delete a version only once you're certain no stored data
or in-flight message still uses it.
