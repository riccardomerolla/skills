---
name: zen-of-ricky
description: Ricky's guiding principles for writing excellent Scala 3 + ZIO code. Use when writing, reviewing, or refactoring Scala code to ensure type safety, immutability, domain-oriented design, and testability. Apply these principles to catch bugs at compile time, eliminate illegal states, and maintain clean, functional architectures.
---

# Zen of Ricky: Guiding Principles for Scala 3 + ZIO

## MAKE ILLEGAL STATES UNREPRESENTABLE

Use compile-time validation to prevent bugs before they happen.

### Use Refined Types

```scala
// Bad: Allows empty strings and invalid values
def createUser(name: String, age: Int): User = ???

// Good: Impossible to create invalid states
import eu.timepit.refined.*
import eu.timepit.refined.api.Refined
import eu.timepit.refined.auto.*
import eu.timepit.refined.numeric.*
import eu.timepit.refined.collection.*

type NonEmptyString = String Refined NonEmpty
type PositiveInt = Int Refined Positive
type Age = Int Refined Interval.Closed[0, 150]

def createUser(name: NonEmptyString, age: Age): User = ???
```

### Use Proper Collection Types

```scala
// Bad: List could be empty, leading to runtime errors
def processItems(items: List[Item]): Result =
  val first = items.head  // Runtime exception if empty!
  ???

// Good: Type guarantees non-emptiness
import zio.prelude.NonEmptyList

def processItems(items: NonEmptyList[Item]): Result =
  val first = items.head  // Always safe!
  ???
```

### Closed Branching with ADTs

```scala
// Exhaustive pattern matching catches missing cases at compile time
enum PaymentStatus:
  case Pending
  case Approved(transactionId: String)
  case Rejected(reason: String)
  case Refunded(amount: BigDecimal)

def handlePayment(status: PaymentStatus): IO[PaymentError, Unit] =
  status match
    case PaymentStatus.Pending => processPending
    case PaymentStatus.Approved(txId) => recordApproval(txId)
    case PaymentStatus.Rejected(reason) => notifyRejection(reason)
    case PaymentStatus.Refunded(amount) => processRefund(amount)
    // Compiler ensures all cases are handled!
```

### Represent Nullability in Types

```scala
// Bad: Null can sneak in
def findUser(id: String): User = ???  // Could return null!

// Good: Optionality is explicit
def findUser(id: UserId): IO[UserError, Option[User]] = ???

// Even better: Use specific error types
enum UserError:
  case NotFound(id: UserId)
  case DatabaseError(cause: Throwable)

def findUser(id: UserId): IO[UserError, User] = ???
```

### Multiversal Equality with CanEqual

```scala
// Prevent nonsensical comparisons at compile time
import scala.language.strictEquality

case class Person(name: String)
case class Cat(name: String)

given CanEqual[Person, Person] = CanEqual.derived
given CanEqual[Cat, Cat] = CanEqual.derived

val person = Person("Alice")
val cat = Cat("Fluffy")

// person == cat  // Compile error! Can't compare Person to Cat
person == person  // OK
```

### Resource Management

```scala
// Bad: Resource leak potential
def readFile(path: String): String =
  val source = scala.io.Source.fromFile(path)
  source.mkString  // Never closes!

// Good: Automatic resource cleanup with ZIO
def readFile(path: String): Task[String] =
  ZIO.acquireReleaseWith(
    ZIO.attempt(scala.io.Source.fromFile(path))
  )(source => 
    ZIO.succeed(source.close())
  )(source =>
    ZIO.attempt(source.mkString)
  )

// Even better: Use ZIO Streams for large files
def streamFile(path: String): ZStream[Any, Throwable, String] =
  ZStream.acquireReleaseWith(
    ZIO.attempt(scala.io.Source.fromFile(path))
  )(source =>
    ZIO.succeed(source.close())
  ).flatMap(source =>
    ZStream.fromIterator(source.getLines())
  )
```

## NO MUTABILITY EVER

Mutable variables create hidden complexity and bugs. Avoid them completely.

### Pure Functions Are the Ideal

```scala
// Bad: Mutates state
var total = 0
def addToTotal(value: Int): Unit =
  total += value  // Side effect!

// Good: Pure function returns new value
def add(current: Int, value: Int): Int =
  current + value

// In ZIO: Use Ref for shared state
def counter: UIO[Ref[Int]] =
  Ref.make(0)

def increment(ref: Ref[Int]): UIO[Int] =
  ref.updateAndGet(_ + 1)
```

### Effects as Values

```scala
// Bad: Direct side effects
def saveUser(user: User): Unit =
  database.save(user)  // Immediate execution!
  logger.info("User saved")  // Another side effect!

// Good: Effects as immutable descriptions
def saveUser(user: User): ZIO[Database & Logger, DBError, Unit] =
  for {
    _ <- Database.save(user)
    _ <- ZIO.logInfo(s"User saved: ${user.id}")
  } yield ()
```

### Testable Side Effects

```scala
// When effect-oriented approach isn't possible, use dependency injection
trait EmailService:
  def send(to: String, subject: String, body: String): Unit

class NotificationService(emailService: EmailService):
  def notifyUser(user: User, message: String): Unit =
    emailService.send(user.email, "Notification", message)

// Test with mock
class MockEmailService extends EmailService:
  var sentEmails: List[(String, String, String)] = Nil
  def send(to: String, subject: String, body: String): Unit =
    sentEmails = sentEmails :+ (to, subject, body)
```

## DATA TYPES & FUNCTIONS SHOULD BE DOMAIN ORIENTED

### Opaque Types for Domain Modeling

```scala
// Bad: Primitive obsession
def processOrder(customerId: String, productId: String, amount: Double): Order = ???

// Good: Domain types with opaque types (Scala 3)
object Domain:
  opaque type CustomerId = String
  object CustomerId:
    def apply(value: String): CustomerId = value
    extension (id: CustomerId)
      def value: String = id
  
  opaque type ProductId = String
  object ProductId:
    def apply(value: String): ProductId = value
    extension (id: ProductId)
      def value: String = id
  
  opaque type Amount = BigDecimal
  object Amount:
    def apply(value: BigDecimal): Amount = value
    extension (amt: Amount)
      def value: BigDecimal = amt

import Domain.*

def processOrder(
  customerId: CustomerId,
  productId: ProductId,
  amount: Amount
): IO[OrderError, Order] = ???

// Now impossible to mix up parameters!
// processOrder(productId, customerId, amount)  // Compile error!
```

### Parse, Don't Validate

```scala
// Bad: Validate and hope
def createEmail(str: String): Email =
  require(str.contains("@"), "Invalid email")
  Email(str)  // Validation is separate from construction

// Good: Parse into typed result
enum EmailError:
  case Empty
  case MissingAtSign
  case InvalidDomain

def parseEmail(str: String): Either[EmailError, Email] =
  if str.isEmpty then
    Left(EmailError.Empty)
  else if !str.contains("@") then
    Left(EmailError.MissingAtSign)
  else if !isValidDomain(str.split("@")(1)) then
    Left(EmailError.InvalidDomain)
  else
    Right(Email(str))  // Safe to construct

// With ZIO
def parseEmail(str: String): IO[EmailError, Email] =
  ZIO.fromEither(parseEmailPure(str))
```

### Smart Constructors

```scala
opaque type Age = Int

object Age:
  def make(value: Int): Either[String, Age] =
    if value >= 0 && value <= 150 then
      Right(value)
    else
      Left(s"Age must be between 0 and 150, got $value")
  
  extension (age: Age)
    def value: Int = age

// Usage
Age.make(25)   // Right(25)
Age.make(-5)   // Left("Age must be between 0 and 150, got -5")
Age.make(200)  // Left("Age must be between 0 and 150, got 200")
```

## TEST ONIONS ARE GOOD. TEST ASSERTION DSLS ARE BAD

### Testing Pyramid with ZIO

**Level 1: Types (Fastest)**
```scala
// Type system catches bugs at compile time
def transfer(from: Account, to: Account, amount: Amount): IO[TransferError, Unit]

// Can't pass wrong types:
// transfer(amount, from, to)  // Compile error!
```

**Level 2: Pure Functions with Unit Tests**
```scala
import zio.test.*

// Pure domain logic
def calculateDiscount(price: BigDecimal, couponCode: String): BigDecimal =
  if couponCode == "SAVE20" then price * 0.8
  else price

object DiscountSpec extends ZIOSpecDefault:
  def spec = suite("Discount calculation")(
    test("applies 20% discount for SAVE20") {
      val result = calculateDiscount(BigDecimal(100), "SAVE20")
      assertTrue(result == BigDecimal(80))
    },
    test("no discount for invalid code") {
      val result = calculateDiscount(BigDecimal(100), "INVALID")
      assertTrue(result == BigDecimal(100))
    }
  )
```

**Level 3: Integration Tests with Fake Implementations**
```scala
// Test layer using in-memory implementation
case class InMemoryUserRepo(users: Ref[Map[UserId, User]]) extends UserRepo:
  def find(id: UserId): IO[RepoError, Option[User]] =
    users.get.map(_.get(id))
  
  def save(user: User): IO[RepoError, Unit] =
    users.update(_ + (user.id -> user))

object InMemoryUserRepo:
  val test: ZLayer[Any, Nothing, UserRepo] =
    ZLayer.fromZIO(
      Ref.make(Map.empty[UserId, User]).map(InMemoryUserRepo(_))
    )

// Test with fake
object UserServiceSpec extends ZIOSpecDefault:
  def spec = suite("UserService")(
    test("creates and retrieves user") {
      for {
        service <- ZIO.service[UserService]
        user = User(UserId("123"), "Alice")
        _ <- service.createUser(user)
        retrieved <- service.getUser(UserId("123"))
      } yield assertTrue(retrieved.contains(user))
    }
  ).provide(UserService.live, InMemoryUserRepo.test)
```

**Level 4: Integration Tests with Testcontainers**
```scala
import com.dimafeng.testcontainers.PostgreSQLContainer

object DatabaseSpec extends ZIOSpecDefault:
  def spec = suite("Database operations")(
    test("stores and retrieves from real PostgreSQL") {
      for {
        repo <- ZIO.service[UserRepo]
        user = User(UserId("123"), "Bob")
        _ <- repo.save(user)
        retrieved <- repo.find(UserId("123"))
      } yield assertTrue(retrieved.contains(user))
    }
  ).provide(
    UserRepo.live,
    PostgresLayer.testcontainer
  )
```

### Use assertTrue with Regular Code

```scala
// Good: Simple, clear boolean assertions
test("user age is valid") {
  val user = User("Alice", 25)
  assertTrue(user.age >= 0 && user.age <= 150)
}

test("list contains expected elements") {
  val items = List(1, 2, 3, 4, 5)
  assertTrue(
    items.length == 5,
    items.contains(3),
    items.head == 1,
    items.last == 5
  )
}

// Avoid custom assertion DSLs
// Bad: items should contain(3)
// Bad: items must haveSize(5)
// Good: assertTrue(items.contains(3), items.length == 5)
```

## Summary Checklist

When writing Scala 3 + ZIO code, ensure:

- [ ] Illegal states are unrepresentable (refined types, ADTs, Option/Either)
- [ ] No `var`, no mutable collections, no mutable state
- [ ] Pure functions for domain logic
- [ ] Effects represented as ZIO values
- [ ] Domain types using opaque types or type aliases
- [ ] Parse inputs into typed values, don't just validate
- [ ] Resources managed with `acquireRelease` or `ZLayer.scoped`
- [ ] Tests progress from types → pure functions → fakes → testcontainers
- [ ] Use `assertTrue` with boolean expressions
- [ ] Pattern matching is exhaustive (compiler-checked)
- [ ] Multiversal equality prevents nonsensical comparisons
