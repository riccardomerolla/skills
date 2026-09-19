# Advanced Patterns: Making Illegal States Unrepresentable

## Phantom Types for State Machines

Encode state transitions in the type system to prevent invalid operations.

```scala
// States as phantom types
sealed trait ConnectionState
sealed trait Disconnected extends ConnectionState
sealed trait Connected extends ConnectionState
sealed trait Authenticated extends ConnectionState

class Connection[S <: ConnectionState] private (underlying: Socket):
  def connect()(using ev: S =:= Disconnected): Connection[Connected] =
    underlying.connect()
    Connection[Connected](underlying)
  
  def authenticate(credentials: Credentials)(using ev: S =:= Connected): Connection[Authenticated] =
    underlying.authenticate(credentials)
    Connection[Authenticated](underlying)
  
  def sendMessage(msg: String)(using ev: S =:= Authenticated): Unit =
    underlying.send(msg)  // Only possible when authenticated!

object Connection:
  def create(): Connection[Disconnected] =
    Connection[Disconnected](Socket.create())

// Usage
val conn1 = Connection.create()
// conn1.sendMessage("hello")  // Compile error! Not authenticated
val conn2 = conn1.connect()
// conn2.sendMessage("hello")  // Compile error! Not authenticated
val conn3 = conn2.authenticate(credentials)
conn3.sendMessage("hello")  // OK!
```

## Indexed Monads for Stateful Workflows

```scala
import zio.prelude.*

// State machine for order processing
sealed trait OrderState
case object Draft extends OrderState
case object Validated extends OrderState
case object Paid extends OrderState
case object Shipped extends OrderState

case class Order[S <: OrderState](
  id: OrderId,
  items: List[Item],
  total: Amount,
  state: S
)

object OrderWorkflow:
  def validate(order: Order[Draft.type]): IO[ValidationError, Order[Validated.type]] =
    ZIO.attempt {
      require(order.items.nonEmpty, "Order must have items")
      require(order.total.value > 0, "Order total must be positive")
      order.copy(state = Validated)
    }.mapError(e => ValidationError(e.getMessage))
  
  def pay(order: Order[Validated.type], payment: Payment): IO[PaymentError, Order[Paid.type]] =
    for {
      _ <- PaymentService.process(payment)
      _ <- ZIO.logInfo(s"Payment processed for order ${order.id}")
    } yield order.copy(state = Paid)
  
  def ship(order: Order[Paid.type]): IO[ShippingError, Order[Shipped.type]] =
    for {
      _ <- ShippingService.schedule(order)
      _ <- ZIO.logInfo(s"Order ${order.id} shipped")
    } yield order.copy(state = Shipped)

// Type-safe workflow
def processOrder(draft: Order[Draft.type]): IO[OrderError, Order[Shipped.type]] =
  for {
    validated <- OrderWorkflow.validate(draft)
    paid <- OrderWorkflow.pay(validated, payment)
    shipped <- OrderWorkflow.ship(paid)
  } yield shipped
```

## Dependent Types for Length-Indexed Collections

```scala
import scala.compiletime.ops.int.*

// Fixed-size vector
sealed trait Nat
case object Zero extends Nat
case class Succ[N <: Nat]() extends Nat

type _0 = Zero.type
type _1 = Succ[_0]
type _2 = Succ[_1]
type _3 = Succ[_2]

case class Vec[N <: Nat, +A](values: List[A]):
  def head(using ev: N =:= Succ[?]): A = values.head
  
  def append[M <: Nat](other: Vec[M, A]): Vec[?, A] =
    Vec(values ++ other.values)

object Vec:
  def empty[A]: Vec[_0, A] = Vec(Nil)
  
  def single[A](a: A): Vec[_1, A] = Vec(List(a))
  
  def apply[A](a1: A, a2: A): Vec[_2, A] = Vec(List(a1, a2))
  
  def apply[A](a1: A, a2: A, a3: A): Vec[_3, A] = Vec(List(a1, a2, a3))

// Usage
val v0 = Vec.empty[Int]
// v0.head  // Compile error! Empty vector has no head

val v1 = Vec.single(1)
v1.head  // OK, guaranteed to have at least one element

val v3 = Vec(1, 2, 3)
v3.head  // OK
```

## Newtypes for Zero-Cost Abstraction

```scala
import io.estatico.newtype.macros.newtype

object Domain:
  @newtype case class UserId(value: String)
  @newtype case class Email(value: String)
  @newtype case class OrderId(value: String)
  
  // Smart constructor with validation
  object Email:
    def parse(str: String): Either[String, Email] =
      if str.matches(".+@.+\\..+") then
        Right(Email(str))
      else
        Left(s"Invalid email: $str")

import Domain.*

// Zero runtime overhead, full type safety
def sendEmail(userId: UserId, email: Email): Task[Unit] = ???

// Can't mix up parameters
val userId = UserId("user-123")
val email = Email.parse("user@example.com").toOption.get
sendEmail(userId, email)  // OK
// sendEmail(email, userId)  // Compile error!
```

## Refined Types for Runtime Validation

```scala
import eu.timepit.refined.*
import eu.timepit.refined.api.Refined
import eu.timepit.refined.auto.*
import eu.timepit.refined.numeric.*
import eu.timepit.refined.string.*
import eu.timepit.refined.collection.*
import eu.timepit.refined.boolean.*

// Positive integers
type PositiveInt = Int Refined Positive
val qty: PositiveInt = 5  // OK
// val invalid: PositiveInt = -1  // Compile error!

// Non-empty strings
type NonEmptyString = String Refined NonEmpty
val name: NonEmptyString = "Alice"  // OK
// val empty: NonEmptyString = ""  // Compile error!

// Bounded values
type Percentage = Int Refined Interval.Closed[0, 100]
val discount: Percentage = 20  // OK
// val invalid: Percentage = 150  // Compile error!

// Email validation
type Email = String Refined MatchesRegex[".+@.+\\..+"]
val email: Email = "user@example.com"  // OK

// Custom predicates
import eu.timepit.refined.api.Validate

case class DivisibleBy(n: Int)
given Validate.Plain[Int, DivisibleBy] =
  Validate.fromPredicate(
    x => x % n == 0,
    x => s"$x is not divisible by $n",
    DivisibleBy(n)
  )

type Even = Int Refined DivisibleBy(2)
val even: Even = 4  // OK
// val odd: Even = 3  // Compile error!

// Runtime refinement with ZIO
def refineAge(value: Int): IO[String, Int Refined Positive] =
  ZIO.fromEither(
    refineV[Positive](value)
  )

def createUser(name: String, ageValue: Int): IO[UserError, User] =
  for {
    nonEmptyName <- ZIO.fromEither(
      refineV[NonEmpty](name)
    ).mapError(msg => UserError.InvalidName(msg))
    
    age <- ZIO.fromEither(
      refineV[Positive](ageValue)
    ).mapError(msg => UserError.InvalidAge(msg))
    
  } yield User(nonEmptyName, age)
```

## Total Functions with Exhaustive Matching

```scala
// Encode all possible outcomes in the return type
sealed trait Result[+A]
case class Success[A](value: A) extends Result[A]
case class Failure(error: String) extends Result[Nothing]
case object NotFound extends Result[Nothing]

def processResult[A, B](result: Result[A])(f: A => B): Option[B] =
  result match
    case Success(value) => Some(f(value))
    case Failure(_) => None
    case NotFound => None
    // Compiler ensures all cases are handled

// With ZIO, use typed errors instead
def operation: IO[AppError, User] = ???

enum AppError:
  case NotFound(id: String)
  case Unauthorized
  case ServiceUnavailable

def handleError(error: AppError): String =
  error match
    case AppError.NotFound(id) => s"User $id not found"
    case AppError.Unauthorized => "Access denied"
    case AppError.ServiceUnavailable => "Service temporarily unavailable"
    // Exhaustive - compiler ensures all cases covered
```

## Builder Pattern with Phantom Types

```scala
sealed trait BuilderState
sealed trait NoName extends BuilderState
sealed trait HasName extends BuilderState
sealed trait NoAge extends BuilderState  
sealed trait HasAge extends BuilderState

case class PersonBuilder[NameState <: BuilderState, AgeState <: BuilderState] private (
  name: Option[String],
  age: Option[Int],
  address: Option[String]
):
  def withName(n: String)(using ev: NameState =:= NoName): PersonBuilder[HasName, AgeState] =
    PersonBuilder[HasName, AgeState](Some(n), age, address)
  
  def withAge(a: Int)(using ev: AgeState =:= NoAge): PersonBuilder[NameState, HasAge] =
    PersonBuilder[NameState, HasAge](name, Some(a), address)
  
  def withAddress(addr: String): PersonBuilder[NameState, AgeState] =
    copy(address = Some(addr))
  
  def build(using 
    ev1: NameState =:= HasName,
    ev2: AgeState =:= HasAge
  ): Person =
    Person(name.get, age.get, address)

object PersonBuilder:
  def create: PersonBuilder[NoName, NoAge] =
    PersonBuilder[NoName, NoAge](None, None, None)

// Usage
val person = PersonBuilder.create
  .withName("Alice")  // OK
  .withAge(30)        // OK
  // .withAge(35)     // Compile error! Age already set
  .withAddress("123 Main St")
  .build              // OK - all required fields set

// val invalid = PersonBuilder.create.build  // Compile error! Missing name and age
```

## Session Types for Protocol Compliance

```scala
// Ensure protocols are followed at compile time
sealed trait Protocol
sealed trait Init extends Protocol
sealed trait Ready extends Protocol
sealed trait Running extends Protocol
sealed trait Stopped extends Protocol

case class Session[S <: Protocol] private (state: S)

object Session:
  def start(): Session[Init] = Session(Init)
  
  extension (s: Session[Init])
    def initialize: IO[InitError, Session[Ready]] =
      ZIO.attempt {
        // Initialization logic
        Session[Ready](Ready)
      }.mapError(e => InitError(e.getMessage))
  
  extension (s: Session[Ready])
    def run: IO[RunError, Session[Running]] =
      ZIO.attempt {
        // Start execution
        Session[Running](Running)
      }.mapError(e => RunError(e.getMessage))
  
  extension (s: Session[Running])
    def stop: IO[StopError, Session[Stopped]] =
      ZIO.attempt {
        // Cleanup
        Session[Stopped](Stopped)
      }.mapError(e => StopError(e.getMessage))

// Type-safe protocol
def workflow: IO[SessionError, Session[Stopped]] =
  for {
    init <- ZIO.succeed(Session.start())
    ready <- init.initialize
    running <- ready.run
    stopped <- running.stop
  } yield stopped
```
