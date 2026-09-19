# Advanced ZIO Patterns

This reference contains advanced patterns and detailed guidance for complex ZIO scenarios.

## Advanced Error Handling

### Error Hierarchies

```scala
// Subsystem-specific errors
sealed trait SigningError
object SigningError:
  case class InvalidKey(msg: String) extends SigningError
  case class NetworkFailure(cause: Throwable) extends SigningError
  case object Timeout extends SigningError

sealed trait LiquidityError
object LiquidityError:
  case class InsufficientFunds(required: BigDecimal, available: BigDecimal) extends LiquidityError
  case class PriceSlippage(expected: BigDecimal, actual: BigDecimal) extends LiquidityError
  case class Unexpected(cause: Throwable) extends LiquidityError

// Wiring-level errors that compose subsystems
sealed trait ApplicationError
object ApplicationError:
  case class Signing(error: SigningError) extends ApplicationError
  case class Liquidity(error: LiquidityError) extends ApplicationError
  case class Configuration(msg: String) extends ApplicationError
```

### Error Mapping at Boundaries

```scala
// Map subsystem errors to wiring errors at composition points
val application: ZLayer[Any, ApplicationError, Application] =
  ZLayer.make[Application](
    SigningService.live.mapError(ApplicationError.Signing(_)),
    LiquidityService.live.mapError(ApplicationError.Liquidity(_)),
    ConfigService.live.mapError(e => ApplicationError.Configuration(e.getMessage))
  )
```

### Union Type Errors (Scala 3)

```scala
// For small, explicit error sets
type ServiceError = NotFound | RateLimited | Unauthorized

def fetch(id: String): IO[ServiceError, Data] =
  ZIO.attempt(...)
    .mapError {
      case _: NoSuchElementException => NotFound(id)
      case _: RateLimitException => RateLimited()
      case _: AuthException => Unauthorized()
    }
```

## Advanced ZLayer Patterns

### Polymorphic Services

```scala
trait Cache[F[_, _, _]]:
  def get[E, A](key: String): F[Any, Option[E], Option[A]]
  def put[E, A](key: String, value: A): F[Any, E, Unit]

final case class CacheLive[F[_, _, _]: ZIO](
  store: Ref[Map[String, Any]]
) extends Cache[F]:
  def get[E, A](key: String): F[Any, Option[E], Option[A]] =
    store.get.map(_.get(key).map(_.asInstanceOf[A]))
  
  def put[E, A](key: String, value: A): F[Any, E, Unit] =
    store.update(_ + (key -> value))

object Cache:
  val live: ZLayer[Any, Nothing, Cache[IO]] =
    ZLayer.fromZIO(
      Ref.make(Map.empty[String, Any]).map(CacheLive(_))
    )
```

### Scoped Layers for Background Work

```scala
object MetricsUpdater:
  val live: ZLayer[MetricsClient, Nothing, Unit] =
    ZLayer.scoped {
      for {
        client <- ZIO.service[MetricsClient]
        _ <- updateLoop(client)
          .forkScoped  // Fiber bound to layer's scope
      } yield ()
    }
  
  private def updateLoop(client: MetricsClient): Task[Nothing] =
    (for {
      metrics <- collectMetrics
      _ <- client.send(metrics)
      _ <- ZIO.sleep(30.seconds)
    } yield ()).forever
```

### Layer Composition with provideSomeLayer

```scala
// Narrow environment instead of widening dependencies
def businessLogic: ZIO[UserRepo & PaymentService, AppError, Result] = ???

val program: ZIO[Database & HttpClient, AppError, Result] =
  businessLogic.provideSomeLayer[Database & HttpClient](
    UserRepo.live ++ PaymentService.live
  )
```

## Advanced Concurrency

### Bounded Parallelism with Resource Pooling

```scala
def processWithPool[A, B](
  items: List[A],
  poolSize: Int
)(process: A => Task[B]): Task[List[B]] =
  for {
    semaphore <- Semaphore.make(poolSize)
    results <- ZIO.foreachPar(items) { item =>
      semaphore.withPermit(process(item))
    }
  } yield results
```

### Rate Limiting

```scala
import zio.RateLimiter

val rateLimiter: ZIO[Scope, Nothing, RateLimiter] =
  RateLimiter.make(max = 100, duration = 1.second)

def callAPI[A](request: Request): Task[A] =
  ZIO.scoped {
    for {
      limiter <- rateLimiter
      _ <- limiter.await(1)
      response <- makeRequest(request)
    } yield response
  }
```

### Supervision and Fiber Management

```scala
def supervisedWork: Task[Unit] =
  ZIO.scoped {
    for {
      supervisor <- Supervisor.track
      _ <- ZIO.foreachParDiscard(tasks) { task =>
        task.forkIn(supervisor.value)
      }
      _ <- monitorFibers(supervisor)
    } yield ()
  }

def monitorFibers(supervisor: Supervisor[Set[Fiber.Runtime[Any, Any]]]): Task[Unit] =
  supervisor.value.flatMap { fibers =>
    ZIO.foreach(fibers)(_.await).unit
  }
```

### STM for Composable Concurrency

```scala
import zio.stm.*

case class Account(balance: TRef[BigDecimal])

def transfer(from: Account, to: Account, amount: BigDecimal): UIO[Unit] =
  STM.atomically {
    for {
      fromBalance <- from.balance.get
      _ <- STM.check(fromBalance >= amount)
      _ <- from.balance.update(_ - amount)
      _ <- to.balance.update(_ + amount)
    } yield ()
  }
```

## Advanced Stream Patterns

### Chunk-Aware Processing

```scala
def processChunks[A, B](
  stream: ZStream[Any, Throwable, A],
  chunkSize: Int
)(process: Chunk[A] => Task[Chunk[B]]): ZStream[Any, Throwable, B] =
  stream
    .rechunk(chunkSize)
    .mapChunksZIO(process)
```

### Stream Backpressure

```scala
def processWithBackpressure[A](
  stream: ZStream[Any, Throwable, A],
  bufferSize: Int
)(process: A => Task[Unit]): Task[Unit] =
  for {
    queue <- Queue.bounded[A](bufferSize)
    _ <- stream.runIntoQueue(queue).fork
    _ <- ZStream.fromQueue(queue)
      .mapZIOPar(8)(process)
      .runDrain
  } yield ()
```

### Resource-Safe Streaming

```scala
def streamFromResource[R, A](
  acquire: ZIO[R, Throwable, Source],
  read: Source => Task[Option[A]],
  release: Source => UIO[Unit]
): ZStream[R, Throwable, A] =
  ZStream.acquireReleaseWith(acquire)(release)
    .flatMap { source =>
      ZStream.repeatZIOOption(read(source))
    }
```

## Advanced Testing

### Property-Based Testing

```scala
import zio.test.*

test("commutative property") {
  check(Gen.int, Gen.int) { (a, b) =>
    assertTrue(add(a, b) == add(b, a))
  }
}

// Custom generators with shrinking
val validUserGen: Gen[Any, User] =
  for {
    id <- Gen.uuid
    name <- Gen.alphaNumericStringBounded(3, 20)
    age <- Gen.int(18, 100)
  } yield User(id, name, age)
```

### Testing Time-Based Logic

```scala
test("retries with exponential backoff") {
  for {
    ref <- Ref.make(0)
    fiber <- failingEffect(ref)
      .retry(Schedule.exponential(100.millis) && Schedule.recurs(3))
      .fork
    _ <- TestClock.adjust(100.millis)
    _ <- TestClock.adjust(200.millis)
    _ <- TestClock.adjust(400.millis)
    result <- fiber.join.exit
  } yield assertTrue(result.isSuccess)
}
```

### Testing Concurrent Behavior

```scala
test("parallel processing maintains order") {
  for {
    results <- ZIO.foreachPar(1 to 100)(n => ZIO.succeed(n))
  } yield assertTrue(results == (1 to 100).toList)
}

test("race conditions") {
  for {
    ref <- Ref.make(0)
    fibers <- ZIO.foreachPar(1 to 1000)(_ => ref.update(_ + 1)).fork
    _ <- fibers.join
    count <- ref.get
  } yield assertTrue(count == 1000)
}
```

## Advanced Scheduling

### Complex Schedule Composition

```scala
// Retry with exponential backoff, maximum attempts, and jitter
val complexSchedule: Schedule[Any, Any, Long] =
  Schedule.exponential(100.millis, 2.0) &&
  Schedule.recurs(10) &&
  Schedule.jittered

// Different retry strategies per error type
def smartRetry[R, E, A](
  effect: ZIO[R, E, A],
  classify: E => RetryStrategy
): ZIO[R, E, A] =
  effect.retryOrElse(
    Schedule.recurWhile[E] {
      case e if classify(e) == RetryStrategy.Aggressive =>
        true
      case e if classify(e) == RetryStrategy.Conservative =>
        true
      case _ => false
    },
    (err, _) => ZIO.fail(err)
  )
```

### Schedule Combinators

```scala
// Retry with delay cap
val cappedExponential: Schedule[Any, Any, Duration] =
  Schedule.exponential(100.millis) >>> Schedule.duration.map(_.min(10.seconds))

// Conditional retry based on output
val untilSuccess: Schedule[Any, Either[E, A], Either[E, A]] =
  Schedule.recurUntil[Either[E, A]](_.isRight)
```

## Performance Optimization

### Chunk Size Tuning

```scala
// For I/O-bound operations, larger chunks reduce overhead
ZStream.fromFile(path)
  .rechunk(8192)  // 8KB chunks
  .mapChunksZIO(processChunk)

// For CPU-bound operations, smaller chunks improve parallelism
ZStream.fromIterable(largeList)
  .rechunk(100)
  .mapZIOPar(16)(process)
```

### Fiber Optimization

```scala
// Avoid unbounded fiber creation
ZIO.foreachPar(items)(process)  // Bad: creates one fiber per item

ZIO.foreachPar(items)(process)
  .withParallelism(Runtime.getRuntime.availableProcessors * 2)  // Good
```

### Resource Preallocation

```scala
// Preallocate resources in layers
object ResourcePool:
  val live: ZLayer[Config, Throwable, ResourcePool] =
    ZLayer.scoped {
      for {
        config <- ZIO.service[Config]
        pool <- createPool(config.poolSize)
      } yield ResourcePool(pool)
    }
  
  private def createPool(size: Int): ZIO[Scope, Throwable, Queue[Connection]] =
    for {
      queue <- Queue.bounded[Connection](size)
      _ <- ZIO.foreach(1 to size)(_ => 
        openConnection.withFinalizer(conn => queue.offer(conn))
      )
    } yield queue
```
