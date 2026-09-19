# Advanced patterns: making illegal states unrepresentable in TypeScript

Translations of the patterns the Scala 3 + ZIO version carried. The goal is unchanged: a state the program must never reach should not type-check. Every snippet targets `effect@4` (the `4.0.0-rc` line).

## Phantom-typed state machines

A generic parameter that never appears in the data carries the state. Functions accept only the state they can act on.

```ts
import { Effect } from "effect"

type OrderState = "Draft" | "Submitted" | "Paid"

// The state lives only in the type; runtime data is identical across states
export interface Order<S extends OrderState> {
  readonly id: OrderId
  readonly lines: ReadonlyArray<Line>
  readonly _state?: S
}

export const submit = (order: Order<"Draft">): Effect.Effect<Order<"Submitted">, EmptyOrder> =>
  order.lines.length === 0
    ? Effect.fail(new EmptyOrder({ id: order.id }))
    : Effect.succeed({ ...order, _state: undefined as "Submitted" | undefined })

export const pay = (order: Order<"Submitted">, payment: Payment): Effect.Effect<Order<"Paid">, PaymentFailed> => ...

// pay(draftOrder, payment)  // compile error: Order<"Draft"> is not Order<"Submitted">
```

When each state carries different data, prefer a tagged union (next section) over a phantom: the phantom is for "same shape, different permissions".

## Tagged unions with per-state data

```ts
import { Match, Schema } from "effect"

export const Connection = Schema.Union([
  Schema.TaggedStruct("Disconnected", {}),
  Schema.TaggedStruct("Connecting", { attempt: Schema.Int }),
  Schema.TaggedStruct("Connected", { sessionId: Schema.String, since: Schema.DateTimeUtc })
])
export type Connection = typeof Connection.Type

// sessionId only exists on Connected, so it cannot be read in the wrong state
export const sessionId = (connection: Connection) =>
  Match.value(connection).pipe(
    Match.tag("Connected", ({ sessionId }) => Option.some(sessionId)),
    Match.orElse(() => Option.none())
  )
```

A `status: string` plus `sessionId?: string` is the illegal-state generator this replaces: nothing stops `{ status: "Disconnected", sessionId: "abc" }`.

## Typed builders

A builder that tracks which required fields were set refuses `build()` until all are present.

```ts
type Missing = { readonly host: true; readonly port: true }

class ConfigBuilder<M> {
  private constructor(private readonly host?: string, private readonly port?: number) {}

  static empty(): ConfigBuilder<Missing> {
    return new ConfigBuilder()
  }

  withHost(host: string): ConfigBuilder<Omit<M, "host">> {
    return new ConfigBuilder(host, this.port)
  }

  withPort(port: number): ConfigBuilder<Omit<M, "port">> {
    return new ConfigBuilder(this.host, port)
  }

  // build exists only when nothing is missing
  build(this: ConfigBuilder<{}>): Config {
    return { host: this.host!, port: this.port! }
  }
}

ConfigBuilder.empty().withHost("db").withPort(5432).build() // ok
// ConfigBuilder.empty().withHost("db").build()            // compile error: this is ConfigBuilder<{ port: true }>
```

The non-null assertions are confined to the one method the type system already guards. Reach for this only when a plain `Schema.Class` with required fields cannot express the constraint (for example, fields set across several call sites).

## Protocol steps as types

For a request that must go through steps in order, return a distinct type from each step and accept only the previous one.

```ts
declare const authenticate: (raw: RawRequest) => Effect.Effect<AuthenticatedRequest, AuthError>
declare const authorize: (request: AuthenticatedRequest, action: Action) => Effect.Effect<AuthorizedRequest, Forbidden>
declare const execute: (request: AuthorizedRequest) => Effect.Effect<Response, ExecutionError>

// The only way to obtain an AuthorizedRequest is through authorize, so execute cannot be called early
export const handle = (raw: RawRequest, action: Action) =>
  authenticate(raw).pipe(
    Effect.flatMap((authenticated) => authorize(authenticated, action)),
    Effect.flatMap(execute)
  )
```

Keep the constructors of the intermediate types private to the module that produces them, so the proof cannot be forged.

## Brands for zero-cost distinctions

```ts
import { Schema } from "effect"

export const CustomerId = Schema.String.pipe(Schema.brand("CustomerId"))
export const ProductId = Schema.String.pipe(Schema.brand("ProductId"))

export type CustomerId = typeof CustomerId.Type
export type ProductId = typeof ProductId.Type

declare const reorder: (customer: CustomerId, product: ProductId) => void
// reorder(productId, customerId)  // compile error, even though both are strings at runtime
```

Brands cost nothing at runtime and remove the entire class of swapped-argument bugs. Combine with checks (`Schema.String.check(Schema.isPattern(/^c_/)).pipe(Schema.brand("CustomerId"))`) when the format is also a rule.

## Total functions through exhaustive matching

`Match.exhaustive` fails to compile when a union member is unhandled. A `default` branch or `Match.orElse` turns a total function into a partial one silently, so use them only when the fallback is genuinely correct for every future member.

```ts
import { Match } from "effect"

export const label = (grant: GrantLevel): string =>
  Match.value(grant).pipe(
    Match.when("None", () => "no access"),
    Match.when("Read", () => "read-only"),
    Match.when("Write", () => "read and write"),
    Match.when("Push", () => "can push"),
    Match.exhaustive
  )
```
