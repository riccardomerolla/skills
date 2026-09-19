# Quick Reference: Common ZIO Idioms

## Effect Creation

```scala
// Pure values
ZIO.succeed(value)
ZIO.fail(error)
ZIO.none  // ZIO.succeed(None)
ZIO.some(value)  // ZIO.succeed(Some(value))

// Effectful operations
ZIO.attempt(riskyOperation())
ZIO.attemptBlocking(blockingIO())
ZIO.async[R, E, A](callback => ...)
ZIO.fromFuture(implicit ec => future)

// Conditional effects
ZIO.when(condition)(effect)
ZIO.unless(condition)(effect)
ZIO.ifZIO(condition)(onTrue, onFalse)
```

## Sequencing

```scala
// Sequential
effect1 *> effect2  // Run both, keep second result
effect1 <* effect2  // Run both, keep first result
effect1.as(newValue)  // Replace success value

// Parallel
effect1.zipPar(effect2)
effect1.raceFirst(effect2)  // Cancel loser
effect1.race(effect2)  // Keep first to complete

// For-comprehension
for {
  a <- effectA
  b <- effectB(a)
  c <- effectC(a, b)
} yield c
```

## Error Handling

```scala
// Catching
effect.catchAll(handleError)
effect.catchSome { case SpecificError => recover }
effect.orElse(fallback)
effect.orDie  // Convert to defect

// Mapping
effect.mapError(transform)
effect.mapBoth(transformError, transformSuccess)

// Inspecting
effect.tapError(logError)
effect.tapErrorCause(logCause)

// Recovery
effect.fold(handleError, handleSuccess)
effect.foldZIO(handleError, handleSuccess)
```

## Retries & Timeouts

```scala
// Retry
effect.retry(Schedule.recurs(3))
effect.retry(Schedule.exponential(100.millis))
effect.retryN(5)

// Timeout
effect.timeout(10.seconds)
effect.timeoutFail(CustomTimeout())(10.seconds)

// Combination
effect
  .timeout(30.seconds)
  .retry(Schedule.exponential(1.second) && Schedule.recurs(3))
```

## Resource Management

```scala
// Scoped resources
ZIO.scoped {
  for {
    resource <- ZIO.acquireRelease(open)(close)
    result <- use(resource)
  } yield result
}

// With interruption handling
ZIO.acquireReleaseInterruptible(acquire)(release)

// In layers
ZLayer.scoped(acquireResource)
```

## Parallelism

```scala
// Bounded parallelism
ZIO.foreachPar(items)(process).withParallelism(8)
ZIO.collectAllPar(effects).withParallelism(4)

// Unbounded (use carefully)
ZIO.foreachPar(items)(process)
effects.zipAllPar

// Discard results
ZIO.foreachParDiscard(items)(process)
```

## Collections

```scala
// Traverse
ZIO.foreach(items)(item => effect(item))
ZIO.collectAll(effects)

// Parallel traverse
ZIO.foreachPar(items)(item => effect(item))
ZIO.collectAllPar(effects)

// Discard results
ZIO.foreachDiscard(items)(item => effect(item))

// Filter
ZIO.filter(items)(item => predicate(item))
ZIO.filterPar(items)(item => predicate(item))

// Partition
ZIO.partition(items)(item => effect(item))
```

## Logging

```scala
ZIO.log("message")
ZIO.logDebug("debug message")
ZIO.logInfo("info message")
ZIO.logWarning("warning")
ZIO.logError("error")
ZIO.logErrorCause("error", cause)

// With context
effect.tapBoth(
  err => ZIO.logError(s"Failed: $err"),
  res => ZIO.logInfo(s"Success: $res")
)
```

## Service Access

```scala
// Access service
ZIO.service[MyService]
ZIO.serviceWith[MyService](_.method)
ZIO.serviceWithZIO[MyService](_.effectMethod)

// Provide dependencies
effect.provide(layer)
effect.provideSome[Remaining](layer)
effect.provideLayer(layer)
```

## Fibers

```scala
// Fork
effect.fork  // Returns Fiber
effect.forkDaemon  // Runs on runtime
effect.forkScoped  // Bound to scope
effect.forkIn(scope)  // Explicit scope

// Join
fiber.join
fiber.await  // Returns Exit
fiber.interrupt

// Supervise
effect.supervised
```

## References

```scala
// Ref (atomic reference)
Ref.make(initial).flatMap { ref =>
  ref.get *>
  ref.set(newValue) *>
  ref.update(_ + 1) *>
  ref.updateAndGet(_ * 2)
}

// RefM (effectful operations)
RefM.make(initial).flatMap { ref =>
  ref.updateZIO(current => effectfulCompute(current))
}
```

## Queues

```scala
// Create
Queue.bounded[A](capacity)
Queue.unbounded[A]
Queue.sliding[A](capacity)  // Drops oldest
Queue.dropping[A](capacity)  // Drops newest

// Operations
queue.offer(item)
queue.take  // Blocks until available
queue.takeAll
queue.poll  // Non-blocking, returns Option
queue.size
```

## Promises

```scala
// Create and complete
for {
  promise <- Promise.make[E, A]
  _ <- computation.to(promise).fork
  result <- promise.await
} yield result

// Complete from outside
promise.succeed(value)
promise.fail(error)
promise.complete(exit)
```

## Environment Narrowing

```scala
// Eliminate dependencies
effect.provideSomeLayer[Remaining](layer)
effect.provideEnvironment(env)

// Access partial environment
def needsSubset: ZIO[A & B, E, R] = ???
val provided: ZIO[A, E, R] = needsSubset.provideSomeLayer(BLayer.live)
```

## Common Patterns

```scala
// Run forever
effect.forever

// Repeat with schedule
effect.repeat(Schedule.spaced(1.second))

// Tap (side effect without changing result)
effect.tap(value => ZIO.log(value.toString))

// Ensure (always runs, even on error/interrupt)
effect.ensuring(cleanup)

// Race all
ZIO.raceAll(first, rest)

// Merge errors and success
effect.merge  // ZIO[R, Nothing, Either[E, A]]

// Absolve (unwrap Either)
effectReturningEither.absolve  // ZIO[R, E, A]

// Validate (early termination on first error)
ZIO.validate(items)(item => validate(item))

// ValidatePar (parallel validation, all errors)
ZIO.validatePar(items)(item => validate(item))
```

## Type Signatures

```scala
// Basic
ZIO[R, E, A]  // Requires R, fails with E, succeeds with A

// Common aliases
type Task[A] = ZIO[Any, Throwable, A]
type IO[E, A] = ZIO[Any, E, A]
type UIO[A] = ZIO[Any, Nothing, A]
type RIO[R, A] = ZIO[R, Throwable, A]
type URIO[R, A] = ZIO[R, Nothing, A]

// Layers
ZLayer[RIn, E, ROut]  // Dependency graph node

// Streams
ZStream[R, E, A]  // Stream of A values

// Hub & Queue
Hub[A]  // Publish-subscribe
Queue[A]  // FIFO queue
```

## Exit & Cause

```scala
// Run and inspect exit
effect.run.flatMap {
  case Exit.Success(value) => ZIO.succeed(value)
  case Exit.Failure(cause) => handleCause(cause)
}

// Cause inspection
cause.failures  // List[E]
cause.defects  // List[Throwable]
cause.interrupted  // Boolean

// Exit fold
exit.foldExit(handleFailure, handleSuccess)
```
