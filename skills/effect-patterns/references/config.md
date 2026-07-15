# Runtime configuration with `Config`

Read runtime configuration through Effect's `Config` recipes and provider
layers, never via direct `process.env` access inside application logic. A
`Config<T>` is a yieldable description of "a value from the current
`ConfigProvider`" — which means tests can swap the provider instead of mutating
the environment, and malformed values fail loudly at decode time instead of
propagating `undefined` through the app.

## Reading config inside a layer

Config reads belong in layers (startup/boundary), not in business workflows:

```ts
export const layerFromEnvironment = Layer.effect(
  Configuration.Service,
  Effect.gen(function* () {
    const apiKey = yield* Config.redacted("API_KEY")
    const optionalModel = yield* Config.option(Config.string("MODEL"))
    const enabled = yield* Config.boolean("FEATURE_ENABLED").pipe(
      Config.withDefault(false),
    )
    return Configuration.Service.of({ apiKey, optionalModel, enabled })
  }),
)
```

## The recipe vocabulary

Each combinator encodes a distinct policy — pick the one that says what you mean:

| Recipe | Meaning |
| --- | --- |
| `Config.redacted("KEY")` | credential — value is wrapped so it can't leak into logs |
| `Config.schema(SomeSchema, "KEY")` | refined/validated value (branded paths, ports, URLs) |
| `Config.option(...)` | absence is semantic ("no model configured") |
| `Config.withDefault(v)` | *missing* data gets a default — malformed values still fail |
| `Config.orElse(...)` | intentionally swallow any parse failure (rare; be sure) |
| `Config.unwrap(...)` / `Config.Wrap<T>` | build `layerConfig(...)` helpers (below) |

The `withDefault` vs `orElse` distinction is the load-bearing one: a typo'd
`FEATURE_ENABLED=ture` should fail startup, not silently become `false`.

## Providers

The default provider is `ConfigProvider.fromEnv()`. Replace or extend it with
layers rather than mutating the environment:

- `ConfigProvider.layer(provider)` — replace the active provider for an app or
  test suite.
- `ConfigProvider.layerAdd(provider)` — add a fallback source; pass
  `{ asPrimary: true }` when the added provider must win.
- `ConfigProvider.fromUnknown({...})` — deterministic in-memory config for tests.
- `ConfigProvider.constantCase` — camelCase keys in code read
  `SCREAMING_SNAKE_CASE` env vars.
- `ConfigProvider.nested("PREFIX")` — scope a provider under a prefix.

Treat `.env`, directory, and environment providers as startup/boundary sources.
If a business workflow needs a config value, a layer should have read it and
put it on a service.

## `layer` + `layerConfig` for library-style services

Library-style layers often expose both a concrete `layer(options)` and a
config-backed `layerConfig(options: Config.Wrap<Options>)` — tests pass
concrete values, production wires from env:

```ts
export const layerConfig = (config: Config.Wrap<ClientOptions>) =>
  Layer.effect(
    Client.Service,
    Config.unwrap(config).pipe(
      Effect.flatMap(makeClient),
      Effect.map((client) => Client.Service.of(client)),
    ),
  )
```

## Config in tests

Two levels, depending on what the test exercises:

- `ConfigProvider.layer(ConfigProvider.fromUnknown({...}))` — when the test
  should exercise Config decoding itself.
- `Layer.succeed(AppConfiguration.Service, testConfig)` — when the app wraps
  decoded config in its own service and the test only needs values. See
  [testing.md](testing.md).
