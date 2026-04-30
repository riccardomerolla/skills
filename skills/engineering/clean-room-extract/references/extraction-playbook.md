# Extraction Playbook

Techniques for reconstructing behavior when the source is poorly documented or large. Use these in step 1 (Inventory) and step 3 (Map deep modules) when the README and docs aren't enough.

## When docs are sparse

### Read the public types first

Public types are the contract. Method signatures, exported interfaces, and exported data structures define what the system promises. Build `02-public-api.md` from the type signatures alone, restated in pseudo-syntax. This gives you the skeleton; behaviors get fleshed out from there.

### Read the tests, not the implementation

Tests describe expected behavior in input → output form, which is exactly what `07-test-contracts.md` captures. Read tests and rewrite each as a black-box scenario:

```
Scenario: <one-line description of what's being verified>
Given:    <preconditions, in plain English, no source-language syntax>
When:     <the trigger>
Then:     <the expected observable effect or output>
```

Do **not** copy the test's name or its assertion message verbatim — paraphrase. Do not reference the test framework's idioms (`describe`/`it`/`should`).

If a test is testing implementation details (e.g. "the cache is hit twice"), drop it. The CSP captures externally observable behavior only. If the same property can be expressed externally ("the operation completes in O(1) for repeated inputs"), use that form.

### Infer error semantics from the type/error channel

Languages with explicit error types declare possible failures in the signature:

- Scala/ZIO: `IO[E, A]` — `E` is the error channel
- Rust: `Result<T, E>`
- Haskell: `Either E A`, `ExceptT E m A`
- TypeScript: discriminated unions, `Result<T, E>` libraries
- Go: the second return value
- Java/Kotlin: checked exceptions or `Result`/`Either` types

Catalogue these in `05-effects-and-errors.md` as "the operation may fail with: \<list of failure modes\>", described semantically (`InsufficientFunds`, `ConcurrentModification`) — not by copying the source's exception class names.

### Effect catalogues

If the source uses an effect system (ZIO, cats-effect IO, Effect-TS, Reader/Writer/State stacks), each effect tag tells you what kind of impurity to document: env reads, logging, async I/O, database access, clock reads, randomness. List them in `05-effects-and-errors.md` as **capabilities the build team's runtime must provide**, not as ZLayers, monads, or specific runtime primitives.

For Riccardo's context (Scala 3 + ZIO): the build team may or may not use ZIO. Document "this operation requires a logging capability and read access to a configuration store" — not "uses `ZIO.serviceWithZIO[Logger]`".

### Concurrency and resource semantics

Look for: locks, semaphores, scoped resources (`Resource`, `Scope`, `with`-blocks, `defer`), fiber/goroutine spawning, cancellation, supervision strategies. Document them as **required guarantees**:

- "Operation X must be cancelled when its scope closes."
- "At most N concurrent invocations of Y are permitted system-wide."
- "Resource Z is acquired before first use and released no later than process shutdown."

Not as specific runtime primitives.

## When the source is large

### Layer by layer, outside-in

1. Start at the entry points (CLI commands, HTTP routes, exported library API, message-queue handlers, scheduled jobs).
2. Document those interfaces fully before going inward.
3. Only descend into a deeper module if its behavior can't be inferred from the layer above.

This produces a CSP that's deepest where the public contract is, and lightest where internals are — which is what the build team needs.

### Module map first, then per-module specs

Sketch `01-architecture.md` before any other CSP file. Once the module boundaries are clear, the per-module sections become tractable. Don't write `02-public-api.md` per source file — write it per **deep module**.

A useful heuristic: if two source files together expose a single coherent contract to the rest of the system, they're one module in the CSP. If one source file exposes two unrelated contracts, it's two modules.

### Identify deep modules

A deep module hides a lot of complexity behind a small interface. Symptoms of a deep module worth extracting:

- Many internal types and functions, few exported ones
- The exported interface is stable across the source's git history while the internals churn
- Other modules depend only on the public surface, not on internal types

Symptoms of a shallow module that should be merged or skipped:

- The public surface is essentially a passthrough to one or two internal calls
- Used by exactly one caller
- Mostly type aliases and re-exports

## When you encounter a foreign concept

The source may use a domain term you don't know. Two moves:

1. **Look it up.** If it's a standard term (`vector clock`, `CRDT`, `eventual consistency`, `event sourcing`, `bulkhead`), use the standard definition in `04-domain-glossary.md` without referencing the source's wording. Cite the canonical source if useful.
2. **Rename if proprietary.** If it's a project-specific term (`MalusCorpFlow`, `RippleEngine`, an in-house jargon term), give it a neutral name (`ApprovalFlow`, `RetryEngine`) and record the mapping. The build team works with the new name throughout.

## When data shapes are inferred from runtime, not types

Dynamic-language repos (Python, Ruby, plain JS) often don't expose static types. Reconstruct shapes from:

- Test fixtures and example data
- JSON Schema files, OpenAPI specs, GraphQL SDL
- Docstrings that document arguments
- Validation libraries (`pydantic`, `zod`, `joi`, `dry-validation`) — these often hold the schema even when types don't

Express the inferred shape in the CSP's neutral notation, mark the field as "optional" or "required" based on the validation, and note where you inferred it from at a high level (e.g. "shape inferred from request validators") — without quoting the validators themselves.

## When in doubt

Prefer a smaller, more accurate CSP over a larger, speculative one. Each behavior you describe is a contract the build team will implement. Don't overspecify what the source happens to do; specify what it must do.

When you genuinely cannot determine whether a behavior is intentional or incidental, capture it as observed in `03-behaviors.md` and add a flag like `[INTENT UNCLEAR — build team to decide]`. The build team can choose to preserve, change, or drop it. That's better than a confidently wrong specification.
