---
name: effect-ts-conventions
description: "Use when writing, reviewing, or setting up TypeScript in a repository that uses Effect 4 and the question is repository mechanics: how services, layers, and errors are declared, how modules import and export, how tests fake dependencies, and what CI must pass. Also use when Effect 3 API names (Context.Tag, Data.TaggedError, Schema.decodeUnknown, Schema.minLength) are about to be written from memory. Assumes the official effect-ts skill from Effect-TS/skills is installed."
---

# Effect-TS conventions

Repo-level rules for an Effect 4 codebase: the shape every module, error, fake, and test takes, so that agents extend existing seams instead of inventing parallel ones. Extracted from [llm4ts](https://github.com/riccardomerolla/llm4ts), the reference implementation.

**Prerequisite.** Install the official Effect skill with `npx skills add Effect-TS/skills` and run it once per repo. It installs `effect` and writes the pointer to `node_modules/effect/AGENTS.md`. Read that guide completely before writing Effect code; it owns the API, and everything here defers to it. For how code should be shaped (domain types, illegal states, the test onion), call the Skill tool with "zen-of-ricky".

Every snippet targets `effect@4` (the `4.0.0-rc` line) and was checked against its source.

## Do not write Effect 3 from memory

The single most common failure is a v3 spelling that no longer exists, or still exists with a different meaning. Look it up in `node_modules/effect/src` before trusting your memory. The ones that bite:

| You may remember (v3) | Write instead (v4) |
|---|---|
| `Context.Tag`, `Context.GenericTag`, `Effect.Service` | `Context.Service<Self, Shape>()("id")` |
| `Data.TaggedError` (still exists, but has no schema) | `Schema.TaggedError<Self>()("Tag", { fields })`, so the error encodes and decodes like any other value |
| `Schema.decodeUnknown`, `ParseResult.ParseError` | `Schema.decodeUnknownEffect`, `SchemaError` |
| `Schema.minLength(1)`, `Schema.int()`, `Schema.between(a, b)` | `Schema.String.check(Schema.isNonEmpty())`, `Schema.Int`, `Schema.Int.check(Schema.isBetween({ minimum, maximum }))` |
| `Schema.optional(S)` for "may be absent" | `Schema.optionalKey(S)`: the key may be missing. `Schema.optional` still exists and means the value may also be `undefined`; pick the one you mean |
| `Schema.Literal("a", "b")` | `Schema.Literals(["a", "b"])`; `Schema.Literal` now takes exactly one literal |
| `Schema.Union(A, B)` | `Schema.Union([A, B])` |
| `Effect.catchAll` | `Effect.catch` |
| `Effect.fork` | `Effect.forkChild` or `Effect.forkScoped` |

## Non-negotiable rules

- Every replaceable dependency is a service declared with `Context.Service` and provided by a `Layer`. Nothing reaches for a global, a singleton, or `process.env` directly.
- Every expected failure is a `Schema.TaggedError`. No `throw` in domain code, no global `Error` in an `E` channel, no `unknown` catch that swallows.
- Schemas sit at every external and persistence boundary: CLI arguments, HTTP bodies, files on disk, environment, provider responses. Internal calls pass typed values and never re-decode.
- No `any`, no unchecked `as` assertions, no TypeScript `namespace`, no unmanaged `Promise` (wrap with `Effect.tryPromise` or `Effect.promise`).
- Relative imports end in `.ts`; package imports carry no extension; packages expose explicit subpath exports.
- Secrets never appear in process arguments, logs, traces, persisted state, or error messages.
- Tests are deterministic `@effect/vitest` tests against in-source fakes. The default CI run needs no network, no credentials, and no installed third-party CLI.

## Services and layers

One file per service. The class is the tag, the shape is the interface, and the implementations are named for what they are.

```ts
// packages/core/src/RateLimiter.ts
import { Context, Effect, Layer, Schema } from "effect"

export interface RateLimiterShape {
  readonly acquire: Effect.Effect<void, RateLimiterError>
  readonly tryAcquire: Effect.Effect<boolean>
}

export class RateLimiter extends Context.Service<RateLimiter, RateLimiterShape>()(
  "@llm4ts/core/RateLimiter" // package name plus path, so ids never collide
) {}

export const makeRateLimiter = Effect.fn("@llm4ts/core/RateLimiter.make")(
  function* (config: RateLimiterConfig): Effect.fn.Return<RateLimiterShape> {
    // build the implementation here
  }
)

export const RateLimiterLive = (config: RateLimiterConfig) =>
  Layer.effect(RateLimiter, makeRateLimiter(config))
```

- `FooLive` is the production layer. `FooNoop` is a `Layer.succeed(Foo, { ... })` that does nothing, for callers that do not care. `makeFakeFoo` is the test double (below).
- Build implementations with `Effect.fn("id")` so spans and stack traces carry the name. Do not wrap a bare `Effect.gen` in an arrow function.
- Compose with `Layer.provide` and `Layer.provideMerge`; a layer that needs configuration takes it as a plain argument, as above, or uses `Layer.unwrap`.
- Access a service inside an effect with `yield* RateLimiter`; never construct one by hand outside its own module.

## Errors

```ts
import { Duration, Schema } from "effect"

export class RateLimitError extends Schema.TaggedError<RateLimitError>()("RateLimitError", {
  retryAfter: Schema.optionalKey(Schema.Duration)
}) {
  get message(): string {
    return this.retryAfter === undefined
      ? "rate limited"
      : `rate limited; retry after ${Duration.format(this.retryAfter)}`
  }
}

export class ProviderError extends Schema.TaggedError<ProviderError>()("ProviderError", {
  message: Schema.String,
  cause: Schema.optionalKey(Schema.Defect())
}) {}

// The module's error set, as a schema and as a type
export const LlmError = Schema.Union([RateLimitError, ProviderError])
export type LlmError = typeof LlmError.Type
```

- One class per failure, fields typed, a `message` getter when the text is derived.
- Wrap an underlying exception as `cause: Schema.Defect()`; never stringify it into `message`.
- Export the module's union so signatures read `Effect.Effect<A, LlmError>` and callers handle it with `Effect.catchTag("RateLimitError", ...)` or `Effect.catchTags({...})`.
- Behaviour that diverges from a pinned reference implementation gets an ADR in `docs/adr/`; an error type is part of behaviour.

## Schemas at boundaries

```ts
import { Effect, Schema } from "effect"

export class RateLimiterConfig extends Schema.Class<RateLimiterConfig>("RateLimiterConfig")({
  requestsPerMinute: Schema.Int.pipe(Schema.withConstructorDefault(Effect.succeed(60))),
  burstSize: Schema.Int.pipe(Schema.withConstructorDefault(Effect.succeed(10)))
}) {}

// Untrusted input: decode once
export const parseConfig = (input: unknown) => Schema.decodeUnknownEffect(RateLimiterConfig)(input)

// Trusted internal construction: make, with defaults applied
export const defaults = RateLimiterConfig.make({})
```

`Schema.Class` for records with identity or defaults, `Schema.Struct` for plain shapes, `Schema.TaggedStruct` inside unions, `Schema.Literals` for closed enums. A `Schema.Class` is the type, the constructor, and the decoder: never write a separate `interface` for the same thing.

## Module layout

- `packages/<name>/src/Thing.ts` exports the service, its errors, its config schema, its `Live`/`Noop` layers, and its fake. The test for it is `packages/<name>/test/Thing.test.ts`.
- `package.json` lists explicit `exports` per subpath (`"./RateLimiter": "./src/RateLimiter.ts"` style); consumers import `@scope/core/RateLimiter`, never a deep relative path across packages.
- `tsconfig` runs strict with `exactOptionalPropertyTypes`, `verbatimModuleSyntax`, `rewriteRelativeImportExtensions`, and the `@effect/language-service` plugin with `floatingEffect`, `missingEffectError`, `missingEffectContext`, `globalErrorInEffectFailure`, and `tryCatchInEffectGen` as errors. Treat those diagnostics as failures, not hints.
- The lint config enforces the import-extension rule; `pnpm lint` is the check.
- Deep modules over parallel code: before adding a variant, find the existing seam (a `make*` factory, a registry table, a flow spine) and extend it. Call the Skill tool with "codebase-design" when the seam is not obvious.

## Tests and fakes

A fake lives in the source file next to the real implementation, so every consumer test shares it. It records what it was asked and answers from a plan.

```ts
// in packages/core/src/ProcessExecutor.ts, beside the real executor
export interface FakeProcessExecutor {
  readonly executor: ProcessExecutorShape
  readonly recorded: Effect.Effect<ReadonlyArray<ProcessInvocation>>
}

export const makeFakeProcessExecutor = Effect.fn("@llm4ts/core/ProcessExecutor.makeFake")(
  function* (plan: FakeProcessPlan = {}): Effect.fn.Return<FakeProcessExecutor> {
    const invocations = yield* Ref.make<ReadonlyArray<ProcessInvocation>>([])
    // ... executor consults plan.responses, appends to invocations
  }
)
```

```ts
// packages/core/test/ProcessExecutor.test.ts
import { assert, describe, it } from "@effect/vitest"
import { Effect } from "effect"
import * as TestClock from "effect/testing/TestClock"

describe("ProcessExecutor", () => {
  it.effect("records the invocation and returns the planned result", () =>
    Effect.gen(function* () {
      const fake = yield* makeFakeProcessExecutor({ responses: plan })
      const result = yield* fake.executor.run(["git", "status"], cwd, {})
      const recorded = yield* fake.recorded
      assert.strictEqual(recorded.length, 1)
      assert.deepStrictEqual(result, expected)
    })
  )

  it.effect("fails with a typed error when the plan has no answer", () =>
    Effect.gen(function* () {
      const fake = yield* makeFakeProcessExecutor()
      const error = yield* Effect.flip(fake.executor.run(["unknown"], cwd, {}))
      assert.strictEqual(error._tag, "ProviderError")
    })
  )
})
```

- `it.effect` for anything that yields an effect; `it` for pure functions. `TestClock.adjust` for time; never a real sleep.
- Assert a failure by `Effect.flip` and checking `_tag` and fields; do not `instanceof` and do not use matcher DSLs.
- Provide layers in tests with `Effect.provide(program, Layer)`; when a layer is expensive and shared, use `it.layer` from `@effect/vitest`.
- Nothing in the default test run opens a socket, reads a credential, or shells out to a real CLI. Anything that must is a separate opt-in suite.

## Verification

Run all four before calling any change done, and paste the output that proves it:

```bash
pnpm typecheck
```

```bash
pnpm lint
```

```bash
pnpm format:check
```

```bash
pnpm test
```

For a publishable package, `pnpm build` and a pack smoke test that resolves every subpath export from an external consumer belong to the same gate.

## Quick reference

| Need | Write |
|---|---|
| A dependency | `class Foo extends Context.Service<Foo, FooShape>()("@scope/pkg/Foo") {}` |
| Its production layer | `export const FooLive = Layer.effect(Foo, makeFoo())` |
| Its no-op layer | `export const FooNoop = Layer.succeed(Foo, { ... })` |
| Its test double | `export const makeFakeFoo = Effect.fn("...")(function* (plan) { ... })` in the same file |
| An expected failure | `class FooError extends Schema.TaggedError<FooError>()("FooError", { ... }) {}` |
| The module's failures | `export const FooErrors = Schema.Union([A, B]); export type FooErrors = typeof FooErrors.Type` |
| Untrusted input | `Schema.decodeUnknownEffect(Schema)(input)` once, at the boundary |
| A record with defaults | `Schema.Class` with `Schema.withConstructorDefault` |
| An effectful function | `Effect.fn("Module.name")(function* (args): Effect.fn.Return<A, E> { ... })` |
| A test | `it.effect("...", () => Effect.gen(function* () { ... assert.strictEqual(...) }))` |

## Common mistakes

- **Writing v3 from memory.** Symptom: `Context.Tag`, `Data.TaggedError`, `Schema.decodeUnknown`. Fix: the table at the top, then `node_modules/effect/src`.
- **A second implementation instead of a seam.** Symptom: `makeGitHubConnector`, `makeGitLabConnector`, each with its own retry and logging. Fix: one `makeApiConnector` factory taking the varying parts as arguments.
- **A fake in the test file.** Symptom: three test files each defining their own in-memory store. Fix: `makeMemoryFooStore` exported from the source file.
- **Decoding twice.** Symptom: a schema decode inside a service that already receives a typed value. Fix: decode at the boundary, trust the type inward.
- **An error that is a string.** Symptom: `Effect.fail("not found")`. Fix: a `Schema.TaggedError` with the id as a field.
- **Tests that need the world.** Symptom: a test that passes locally and fails in CI with a network or auth error. Fix: a fake with a plan, and an opt-in suite for the real thing.
