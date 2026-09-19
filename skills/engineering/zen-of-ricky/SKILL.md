---
name: zen-of-ricky
description: "Use when writing, reviewing, or refactoring TypeScript in an Effect 4 codebase and the question is how the code should be shaped: domain types, nullability, error channels, mutability, or the test onion. Applies whenever a plain string, number, null, thrown Error, or mutable variable is about to stand in for a domain concept. Assumes the official effect-ts skill from Effect-TS/skills is installed."
---

# Zen of Ricky: house principles for TypeScript + Effect 4

The principles are the same ones the Scala 3 + ZIO version held. Only the spelling changed.

**Prerequisite.** Install the official Effect skill with `npx skills add Effect-TS/skills`, run it once per repo, and read `node_modules/effect/AGENTS.md` completely before writing Effect code. That guide owns the API details. This skill owns the judgment calls on top of it; where the two seem to disagree on an API, the guide wins. For repo mechanics (services, layers, imports, tests, CI), call the Skill tool with "effect-ts-conventions".

Every snippet below targets `effect@4` (the `4.0.0-rc` line) and was checked against its source.

## Make illegal states unrepresentable

Move validation to construction time, once, and let the type carry the proof everywhere else.

### Brand and check at the boundary

```ts
import { Schema } from "effect"

// Bad: any string is a UserId, any number is an age
declare const createUserBad: (id: string, age: number) => void

// Good: the schema is the only way in; the brand stops mix-ups
export const UserId = Schema.String.pipe(Schema.brand("UserId"))
export type UserId = typeof UserId.Type

export const Age = Schema.Int.check(Schema.isBetween({ minimum: 0, maximum: 150 }))
export type Age = typeof Age.Type

export const NonEmptyName = Schema.String.check(Schema.isNonEmpty())
```

Decode untrusted input with `Schema.decodeUnknownEffect(schema)(input)`; the failure is a typed `SchemaError`, not a thrown exception. Never re-validate a branded value downstream: if it has the brand, it passed.

### Non-empty is a type, not a comment

```ts
import { Array as Arr, Schema } from "effect"

// Bad: items[0] can be undefined and the compiler cannot tell
declare const firstBad: (items: ReadonlyArray<string>) => string

// Good: the signature refuses the empty case
export const first = (items: Arr.NonEmptyReadonlyArray<string>): string => Arr.headNonEmpty(items)

export const Batch = Schema.NonEmptyArray(Schema.String)
```

### Closed branching, exhaustive matching

```ts
import { Match, Schema } from "effect"

export const PaymentStatus = Schema.Union([
  Schema.TaggedStruct("Pending", {}),
  Schema.TaggedStruct("Approved", { transactionId: Schema.String }),
  Schema.TaggedStruct("Rejected", { reason: Schema.String })
])
export type PaymentStatus = typeof PaymentStatus.Type

export const describe = (status: PaymentStatus): string =>
  Match.value(status).pipe(
    Match.tag("Pending", () => "waiting"),
    Match.tag("Approved", ({ transactionId }) => `approved as ${transactionId}`),
    Match.tag("Rejected", ({ reason }) => `rejected: ${reason}`),
    Match.exhaustive // adding a fourth case fails to compile here
  )
```

`Schema.Literals(["None", "Read", "Write"])` is the closed enum for flat cases. A `string` field that "should be one of" is an open state.

### Absence and failure are in the signature

```ts
import { Effect, Option, Schema } from "effect"

export class UserNotFound extends Schema.TaggedError<UserNotFound>()("UserNotFound", {
  id: Schema.String
}) {}

// Optionality: the caller must handle None
declare const find: (id: UserId) => Effect.Effect<Option.Option<User>>

// Or a typed failure when absence is an error for this caller
declare const get: (id: UserId) => Effect.Effect<User, UserNotFound>
```

No `null`, no `undefined` returns, no `throw` in domain code, no global `Error` in an error channel. Expected failures are `Schema.TaggedError` classes; unexpected ones are defects and stay out of the `E` type.

### Resources release themselves

```ts
import { Effect } from "effect"

// Good: release runs on success, failure, and interruption
export const withHandle = Effect.acquireRelease(openHandle, (handle) => closeHandle(handle))
```

Anything opened inside `Effect.acquireRelease` or a scoped layer is closed for you. A manual `close()` at the end of a function is a leak waiting for the first failure.

## No mutability, ever

```ts
import { Effect, Ref } from "effect"

// Bad: shared mutable state
let total = 0
const addBad = (value: number): void => {
  total += value
}

// Good: pure function, or a Ref when state must be shared across fibers
export const add = (current: number, value: number): number => current + value

export const counter = Effect.gen(function* () {
  const ref = yield* Ref.make(0)
  yield* Ref.update(ref, (n) => n + 1)
  return yield* Ref.get(ref)
})
```

`let`, `push`, and property assignment are the three usual leaks. Use `readonly` fields, `ReadonlyArray`, spread copies, and `Ref` for the state that truly must change.

### Effects are values

An `Effect` describes a computation; nothing runs until the edge of the program (`Effect.runPromise` in the entry point, `it.effect` in tests). Do not call `Effect.run*` inside another effect and do not perform a side effect in `Effect.succeed`; use `Effect.sync` or `Effect.tryPromise`.

## Data types and functions are domain-oriented

### Primitive obsession is the bug

```ts
// Bad: three strings, any order compiles
declare const placeOrderBad: (customerId: string, productId: string, amount: number) => void

// Good: branded types make the swapped-argument call a compile error
declare const placeOrder: (customerId: CustomerId, productId: ProductId, amount: Amount) => void
```

### Parse, don't validate

```ts
import { Effect, Schema } from "effect"

export class User extends Schema.Class<User>("User")({
  id: UserId,
  name: NonEmptyName,
  age: Age,
  email: Schema.optionalKey(Schema.String)
}) {}

// One entry point for untrusted data; the result is a proven User
export const parseUser = (input: unknown) => Schema.decodeUnknownEffect(User)(input)

// Internal construction goes through make, which runs the same checks
export const alice = User.make({ id: UserId.make("u1"), name: "Alice", age: 30 })
```

A validated-but-still-`string` value is a lie: the check happened, but nothing remembers it. A `Schema` type is the memory.

## Test onions are good. Assertion DSLs are bad

Tests progress outward, and each ring is the cheapest place that can catch its class of bug:

| Ring | What it catches | Tool |
|---|---|---|
| 1. Types | Wrong arguments, unhandled cases, unhandled errors | the compiler, plus `@effect/language-service` diagnostics |
| 2. Pure functions | Domain logic | plain `it` with `assert` |
| 3. Fakes | Service wiring, error paths | `it.effect` with in-memory adapters built on `Ref` |
| 4. Real adapters | Integration with the world | opt-in, never in the default CI run |

```ts
import { assert, describe, it } from "@effect/vitest"
import { Effect, Option } from "effect"

describe("UserRegistry", () => {
  it.effect("registers then finds", () =>
    Effect.gen(function* () {
      const registry = yield* makeMemoryUserRegistry()
      yield* registry.register(alice)
      const found = yield* registry.find(alice.id)
      assert.deepStrictEqual(found, Option.some(alice))
    })
  )
})
```

Assert with `assert.strictEqual`, `assert.deepStrictEqual`, and `assert.isTrue` on plain expressions. No `expect(x).toHaveLength(3)` style matchers: they hide the value under a vocabulary that has to be learned, and they do not compose with the types.

For what a fake looks like and how services are provided in tests, call the Skill tool with "effect-ts-conventions".

## Summary checklist

- [ ] Illegal states are unrepresentable: brands, checks, `NonEmptyArray`, closed unions
- [ ] Every `Match` on a union ends in `Match.exhaustive`
- [ ] No `let`, no mutation, `Ref` for shared state
- [ ] Effects are values; `Effect.run*` only at the edge
- [ ] Domain ids and quantities are branded types, never bare primitives
- [ ] Untrusted data enters through one `Schema.decodeUnknownEffect`
- [ ] Absence is `Option`, failure is a `Schema.TaggedError`, never `null` or `throw`
- [ ] Resources go through `Effect.acquireRelease` or a scoped layer
- [ ] Tests move outward through types, pure functions, fakes, then real adapters
- [ ] Assertions are plain `assert.*` calls on plain expressions

## Going deeper

[references/illegal-states-patterns.md](references/illegal-states-patterns.md) translates the advanced patterns (phantom-typed state machines, typed builders, protocol steps) from the Scala version into TypeScript.
