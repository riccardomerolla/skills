---
name: scala3-zio
description: Expert Scala 3 + ZIO 2.x development guidance for effect-oriented programming, functional architectures, type-safe domain modeling, resource management, concurrency, typed error handling, ZLayer dependency injection, and ZIO Test. Use when writing, reviewing, refactoring, or debugging Scala 3 code with ZIO, designing functional architectures, implementing effect systems, managing resources and concurrency, setting up dependency injection with ZLayer, handling typed errors, or writing ZIO tests.
---

# Scala 3 + ZIO 2.x Expert

Produce correct, idiomatic, maintainable, composable, resource-safe ZIO code following effect-oriented programming principles.

## Core Principles

### Effects Are Immutable Descriptions

ZIO values describe computations without executing them. Wrap side effects using:

```scala
ZIO.attempt          // For sync effects that may throw
ZIO.attemptBlocking  // For blocking operations
ZIO.async            // For callback-based async
ZIO.fromFuture       // For Future integration
ZIO.suspend          // For lazy evaluation
```

Never put side effects in `ZIO.succeed` or constructors. Use `ZIO.log*` instead of `println`.

### Typed Error Channels

All errors must be typed domain ADTs. No thrown exceptions in business logic.

```scala
enum SigningError:
  case InvalidKey(msg: String)
  case NetworkFailure(cause: Throwable)
  case Timeout

// Map throwables once at boundaries
effect.mapError {
  case e: IOException => SigningError.NetworkFailure(e)
  case e: TimeoutException => SigningError.Timeout
}
```

Keep domain ADTs per subsystem. Map to higher-level errors at composition boundaries.

### Resource Safety

Manage resources with acquire-release patterns. Finalizers always run (normal, error, interruption).

```scala
ZIO.acquireRelease(acquire)(release)
ZIO.acquireReleaseInterruptible(acquire)(release)
ZLayer.scoped(resource)
```

### Functional Architecture

Domain logic is pure. I/O at edges. No `var`, no shared mutable state. Use `Ref`, `Queue`, `Hub` for concurrency.

## Build & Run

The project uses `sbt` for all tasks:

```bash
sbt compile         # Compile
sbt scalafmtAll    # Format code (run before submitting changes)
sbt assembly       # Build fat JAR
```

## Testing Protocols

**You MUST verify your changes by running tests.**

```bash
sbt test           # Run unit tests (ZIO Test + scalamock-zio, no external secrets)
sbt it:test        # Run integration tests (if needed)
```

## Effect Construction

### Allowed Constructors

```scala
ZIO.succeed(42)                           // Pure success
ZIO.fail(MyError("oops"))                // Pure failure
ZIO.attempt(risky())                      // Sync effect
ZIO.attemptBlocking(blockingIO())        // Blocking I/O
ZIO.async[R, E, A](register)             // Callback-based
ZIO.fromFuture(implicit ec => future)    // Future conversion
ZIO.suspend(ZIO.attempt(...))            // Lazy evaluation
```

### Composition Patterns

```scala
for {
  a <- effectA
  b <- effectB(a)
  _ <- ZIO.log(s"Result: $b")
} yield b

// Parallel composition
a.zipPar(b)
ZIO.foreachPar(items)(process).withParallelism(16)

// Racing
primary.race(fallback)
```

## Error Handling

### Typed, Explicit, Exhaustive

```scala
sealed trait RepoError
case class NotFound(id: String) extends RepoError
case class DatabaseError(cause: Throwable) extends RepoError

effect.catchAll {
  case NotFound(id) => ZIO.succeed(User.default)
  case DatabaseError(e) => ZIO.logError(e.getMessage) *> ZIO.fail(e)
}
```

### Recovery Patterns

```scala
// Retry with exponential backoff
effect.retry(Schedule.exponential(100.millis) && Schedule.recurs(5))

// Fallback
primary.orElse(secondary)

// Timeout
effect.timeout(30.seconds).someOrFail(TimeoutError())

// Hedging
effect.race(effect.delay(p50Latency))
```

## Dependency Injection with ZLayer

### Service Pattern

```scala
trait UserRepo:
  def find(id: UserId): IO[RepoError, User]

final case class UserRepoLive(pool: ConnectionPool) extends UserRepo:
  def find(id: UserId): IO[RepoError, User] =
    ZIO.attemptBlocking(/* query */)
      .mapError(e => DatabaseError(e))

object UserRepo:
  val live: ZLayer[ConnectionPool, Nothing, UserRepo] =
    ZLayer.fromFunction(UserRepoLive.apply)
  
  // Accessor helper
  def find(id: UserId): ZIO[UserRepo, RepoError, User] =
    ZIO.serviceWithZIO[UserRepo](_.find(id))
```

### Wiring Rules

- Define services as pure trait algebras
- Implement in `*Live` case classes
- No side effects in constructors
- Compose layers at application boundary
- Use `ZIO.serviceWithZIO` for access
- Add accessor helpers on companion objects
- Background work must be scoped (`forkScoped`)
- Map edge errors at composition boundaries

### DI Checklist

- [ ] Typed error channels (no raw `Throwable`)
- [ ] Background fibers scoped with `forkScoped`
- [ ] No side effects in constructors
- [ ] Layers defined at module boundaries
- [ ] Servers/loops scoped with `ZLayer.scoped`

## Concurrency

### Structured Concurrency

```scala
// Parallel operations with limit
ZIO.foreachPar(tasks)(run).withParallelism(8)

// Scoped background fiber
ZIO.scoped {
  for {
    fiber <- longRunning.forkScoped
    result <- mainWork
    _ <- fiber.join
  } yield result
}

// Coordination primitives
Queue.bounded[Task](100)
Hub.bounded[Message](1000)
Semaphore.make(permits = 10)
```

### Concurrency Guidelines

- Prefer structured concurrency with scopes
- Use `attemptBlocking` for blocking I/O
- Use coordination primitives over manual locks
- Cancel/supervise long-lived fibers
- Add finalizers for cleanup on interruption

## Testing with ZIO Test

```scala
import zio.test.*

object UserRepoSpec extends ZIOSpecDefault:
  def spec = suite("UserRepo")(
    test("finds existing user") {
      for {
        repo <- ZIO.service[UserRepo]
        user <- repo.find(UserId("123"))
      } yield assertTrue(user.name == "Alice")
    },
    
    test("fails on missing user") {
      for {
        repo <- ZIO.service[UserRepo]
        result <- repo.find(UserId("999")).exit
      } yield assertTrue(result.isFailure)
    }
  ).provide(UserRepo.test)
```

### Test Coverage

- Success cases
- Failure cases  
- Boundary conditions
- Resource cleanup
- Time-based behavior (`TestClock`)
- Concurrency scenarios
- Property-based tests with `Gen.check`

### Test Guidelines

- Use `Gen` + `check` for invariants
- Use `TestClock.adjust` instead of real time
- Keep layers minimal (`ZLayer.succeed`)
- Tear down with `Scope`
- Test streams with varied chunk sizes

## Naming Conventions

- **Effects**: verbs → `loadUser`, `processOrder`
- **Services**: nouns → `UserRepo`, `PaymentService`
- **Layers**: adjectives → `live`, `test`, `mock`
- **Errors**: domain names → `InvalidUserId`, `NetworkTimeout`

## Forbidden Anti-Patterns

❌ `var` or shared mutable state
❌ Blocking (`Thread.sleep`, `Await.result`)
❌ Throwing exceptions for domain errors
❌ Swallowing errors silently
❌ Creating layers inside runtime logic
❌ Side effects in constructors
❌ Using `Throwable` as error type
❌ Printing from business code
❌ Mixing `Future` without conversion

## Common Patterns

### Graceful Shutdown

```scala
ZIO.addFinalizer(fiber.interrupt *> cleanup)
```

### Circuit Breaker

```scala
import dev.rezilience.*

CircuitBreaker.make(
  trippingStrategy = TrippingStrategy.failureCount(maxFailures = 5),
  resetPolicy = ResetPolicy.exponentialBackoff(min = 1.second, max = 1.minute)
).flatMap(breaker => breaker(riskyEffect))
```

### Stream Processing

```scala
ZStream.acquireRelease(acquire)(release)
  .mapChunksZIO(processChunk)
  .via(ZPipeline.rechunk(chunkSize = 1024))
  .runDrain
```

## Output Validation Checklist

Before completing any code generation:

- [ ] Effects use correct `R`, `E`, `A` types
- [ ] Errors are domain ADTs (not `Throwable`)
- [ ] No side effects escape constructors
- [ ] Blocking only in `attemptBlocking`
- [ ] Dependencies via ZLayer
- [ ] Error handling explicit and typed
- [ ] Resource lifecycle guaranteed
- [ ] Parallelism uses safe combinators
- [ ] Idiomatic Scala 3 style
