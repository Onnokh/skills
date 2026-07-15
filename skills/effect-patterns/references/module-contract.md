# The module contract

Every service module follows one skeleton, in one order. The payoff is that once
you've read a single module, you can navigate any other — exports are where you
expect them, named the same thing. Consistency here is worth more than local
cleverness.

## The canonical shape

```ts
// src/account/account.ts
import { Context, Effect, Layer } from "effect"

export interface Interface {
  readonly token: (id: AccountID) => Effect.Effect<string, AccountRepoError>
  readonly list: () => Effect.Effect<ReadonlyArray<Info>, AccountRepoError>
}

export class Service extends Context.Service<Service, Interface>()("@app/Account") {}

export const use = serviceUse(Service)

export const layer = Layer.effect(
  Service,
  Effect.gen(function* () {
    const repo = yield* AccountRepo.Service        // dependencies pulled by type
    const http = yield* HttpClient.HttpClient
    return {
      token: Effect.fn("Account.token")(function* (id) { /* ... */ }),
      list: Effect.fn("Account.list")(function* () { /* ... */ }),
    }
  }),
)

export const defaultLayer = layer.pipe(Layer.provide(AccountRepo.defaultLayer))

export * as Account from "./account"
```

**The fixed order of exports:** `Interface` → `Service` → `use` → `layer` →
`defaultLayer` → self-reexport. Keep it identical in every module so readers can
scan to the part they need without searching.

## Naming rules

- **Service tag string:** `"@<scope>/PascalName"`, e.g. `"@app/Account"`,
  `"@app/SessionStatus"`. The tag is the runtime identity of the service; keep it
  unique and mechanical (scope + the module's PascalCase name). Never reuse a tag.
- **`Effect.fn` trace names:** `"PascalService.method"`, e.g. `"Account.token"`,
  `"SessionStatus.get"`. Special effects (migrations, steps) may use dotted
  suffixes like `"Storage.migration.1"`. These strings show up in traces, so they
  must be stable and predictable.
- **Internal helpers** that aren't part of the public surface use
  `Effect.fnUntraced` — no span, no overhead. Reserve `Effect.fn` for the methods
  you actually want to see in a trace.

## No namespaces, no barrels

Two rules that exist purely to protect tree-shaking and Node's native TypeScript
runner:

- **Never `export namespace Foo { ... }`.** It is not standard ESM, it blocks
  tree-shaking, and Node's type-stripping runner can't execute it. Use flat
  top-level exports plus a self-reexport at the bottom of the file:
  `export * as Foo from "./foo"`. Consumers then get the same ergonomic
  `Foo.Service` / `Foo.layer` namespace projection without the costs.
- **Never add a barrel `index.ts`** to a directory holding several independent
  modules. A barrel forces every import through it, which evaluates *every*
  sibling even when you wanted one. Import the specific file instead:
  `import { SessionRetry } from "@/session/retry"`.

Self-reexport source path:
- File is `foo/foo.ts` → `export * as Foo from "./foo"`
- File is `foo/index.ts` (single-namespace directory) → `export * as Foo from "."`

## The `serviceUse` accessor

Calling a service method normally means `yield* Service` then call the method.
`serviceUse` collapses that into a single call so consumers can write
`Account.use.token(id)`:

```ts
export const serviceUse = <Identifier, Shape>(tag: Context.Service<Identifier, Shape>) => {
  const cache = new Map<string, (...args: unknown[]) => Effect.Effect<unknown, unknown, unknown>>()
  return new Proxy({}, {
    get: (_, key: string) => {
      const cached = cache.get(key)
      if (cached) return cached
      const accessor = (...args: unknown[]) =>
        tag.use((service) => {
          const method = service[key as keyof Shape]
          if (typeof method !== "function")
            return Effect.die(new Error(`Service method not found: ${key}`))
          return (method as (...a: unknown[]) => Effect.Effect<unknown, unknown, unknown>)(...args)
        })
      cache.set(key, accessor)
      return accessor
    },
  }) as ServiceUse<Identifier, Shape>
}
```

Two call styles then coexist, and both are fine:

```ts
const repo = yield* AccountRepo.Service     // pull the whole service — used inside layers
yield* Account.use.token(accountID)          // call one method — used by consumers
```

## File splitting

- **Start as a single file.** A module earns a directory only when it grows.
- **When it grows, split into siblings with no barrel.** A `session/` directory
  might hold `session.ts`, `status.ts`, `retry.ts`, `revert.ts`, … each its own
  self-reexporting namespace.
- **Within a domain directory, split by role:**
  - `schema.ts` — branded IDs, data shapes, error classes. Pure; no `Service`.
  - `repo.ts` — the persistence/database `Service`.
  - main file (`account.ts`) — orchestration that depends on both.

## Allowed deviations (and when)

The skeleton is the default, not a straitjacket. Deviate only with a concrete
reason, and prefer the variant that states that reason structurally:

| Variant | Use when |
| --- | --- |
| `Layer.sync` instead of `Layer.effect` | initialization is pure and has no effectful dependencies (e.g. an in-memory tracker) |
| `Layer.succeed` | the service is a constant value |
| `defaultLayer = Layer.suspend(() => layer.pipe(...))` | the dependency graph is heavy and you want it resolved lazily |
| `layer` is a *function* `(overrides) => Layer...` | the service needs runtime configuration/overrides |
| a module that `provide`s another module's `Service` (no local `Service`) | wrapping a shared service with extra scoping (e.g. per-instance lifecycle) |

If you find yourself deviating without one of these reasons, prefer the canonical
shape.
