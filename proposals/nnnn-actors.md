# Actors

* Proposal: [SE-NNNN](NNNN-actors.md)
* Authors: [John McCall](https://github.com/rjmccall), [Doug Gregor](https://github.com/DougGregor)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: Partial available in [recent `main` snapshots](https://swift.org/download/#snapshots) behind the flag `-Xfrontend -enable-experimental-concurrency`

## Introduction

The [actor model](https://en.wikipedia.org/wiki/Actor_model) involves entities called actors. Each *actor* can perform local computation based on its own state, send messages to other actors, and act on messages received from other actors. Actors run independently, and cannot access the state of other actors, making it a powerful abstraction for managing concurrency in language applications. The actor model has been implemented in a number of programming languages, such as Erlang and Pony, as well as various libraries like Akka (on the JVM) and Orleans (on the .NET CLR).

This proposal introduces a design for *actors* in Swift, providing a model for building concurrent programs that are simple to reason about and are safer from data races.

Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/)

## Motivation

One of the more difficult problems in developing concurrent programs is dealing with [data races](https://en.wikipedia.org/wiki/Race_condition#Data_race). A data race occurs when the same data in memory is accessed by two concurrently-executing threads, at least one of which is writing to that memory. When this happens, the program may behave erratically, including spurious crashes or program errors due to corrupted internal state. 

Data races are notoriously hard to reproduce and debug, because they often depend on two threads getting scheduled in a particular way. 
Tools such as [ThreadSanitizer](https://clang.llvm.org/docs/ThreadSanitizer.html) help, but they are necessarily reactive (as opposed to proactive)--they help find existing bugs, but cannot help prevent them.

Actors provide a model for building concurrent programs that are free of data races. They do so through *data isolation*: each actor protects is own instance data, ensuring that only a single thread will access that data at a given time. Actors shift the way of thinking about concurrency from raw threading to actors and put focus on actors "owning" their local state. This proposal provides a basic isolation model that protects the value-type state of an actor from data races. A full actor isolation model, which protects other state (such as reference types) is left as future work.

## Proposed solution

### Actor classes

This proposal introduces *actor classes* into Swift. An actor class is a form of class that protects access to its mutable state, and is introduced with "actor class":

```swift
actor class BankAccount {
  private let ownerName: String
  private var balance: Double
}
```

Actor classes behave like classes in most respects: they can inherit (from other actor classes), have methods, properties, and subscripts. They can be extended and conform to protocols, be generic, and be used with generics.

The primary difference is that actor classes protect their state from data races. This is enforced statically by the Swift compiler through a set of limitations on the way in which actors and their members can be used, collectively called *actor isolation*.   

### Actor isolation

Actor isolation is how actors protect their mutable state. For actor classes, the primary mechanism for this protection is by only allowing their stored instance properties to be accessed directly on `self`. For example, here is a method that attempts to transfer money from one account to another:

```swift
extension BankAccount {
  enum BankError: Error {
    case insufficientFunds
  }
  
  func transfer(amount: Double, to other: BankAccount) throws {
    if amount > balance {
      throw BankError.insufficientFunds
    }

    print("Transferring \(amount) from \(ownerName) to \(other.ownerName)")

    balance = balance - amount
    other.balance = other.balance + amount  // error: actor-isolated property 'balance' can only be referenced on 'self'
  }
}
```

If `BankAccount` were a normal class, the `transfer(amount:to:)` method would be well-formed, but would be subject to data races in concurrent code without an external locking mechanism. With actor classes, the attempt to reference `other.balance` triggers a compiler error, because `balance` may only be referenced on `self`.

As noted in the error message, `balance` is *actor-isolated*, meaning that it can only be accessed from within the specific actor it is tied to or "isolated by". In this case, it's the instance of `BankAccount` referenced by `self`. Stored properties, computed properties, subscripts, and synchronous instance methods (like `transfer(amount:to:)`) in an actor class are all actor-isolated by default.

On the other hand, the reference to `other.ownerName` is allowed, because `ownerName` is immutable (defined by `let`). Once initialized, it is never written, so there can be no data races in accessing it. `ownerName` is called *actor-independent*, because it can be freely used from any actor. Constants introduced with `let` are actor-independent by default; there is also an attribute `@actorIndependent` (described in [**Actor-independent declarations**](#actor-independent-declarations)) to specify that a particular declaration is actor-independent.

> **Note**: Constants defined by `let` are only truly immutable when the type has value semantics. A `let` that refers to a mutable reference type (such as a non-actor class type with mutable state) would be unsafe based on the rules discussed so far. These issues are discussed later in [**Escaping references**](#escaping-references).

Compile-time actor-isolation checking, as shown above, ensures that code outside of the actor does not interfere with the actor's mutable state. 

Asynchronous function invocations are turned into enqueues of partial tasks representing those invocations to the actor's *queue*. This queue—along with an exclusive task executor bound to the actor—functions as a synchronization boundary between the actor and any of its external callers. For example, if we wanted to make a deposit to a given bank account `account`, we could make a call to a method `deposit(amount:)`, and that call would be placed on the queue. The executor would pull tasks from the queue one-by-one, ensuring an actor never is concurrenty running on multiple threads, and would eventually process the deposit.

Synchronous functions in Swift are not amenable to being placed on a queue to be executed later. Therefore, synchronous instance methods of actor classes are actor-isolated and, therefore, not available from outside the actor instance. For example:

```swift
extension BankAccount {
  func depositSynchronously(amount: Double) {
    assert(amount >= 0)
    balance = balance + amount
  }
}

func printMoney(accounts: [BankAccount], amount: Double) {
  for account in accounts {
    account.depositSynchronously(amount: amount) // error: actor-isolated instance method 'depositSynchronously(amount:)' can only be referenced inside the actor
  }
}
```

It should be noted that actor isolation adds a new dimension, separate from access control, to the decision making process whether or not one is allowed to invoke a specific function on an actor. Specifically, synchronous functions may only be invoked by the specific actor instance itself, and not even by any other instance of the same actor class. 

All interactions with an actor (other than the special-cased access to constants) must be performed asynchronously (semantically, one may think about this as the actor model's messaging to and from the actor). Asynchronous functions provide a mechanism that is suitable for describing such operations, and are explained in depth in the complementary [async/await proposal](https://github.com/DougGregor/swift-evolution/blob/async-await/proposals/nnnn-async-await.md). We can make the `deposit(amount:)` instance method `async`, and thereby make it accessible to other actors (as well as non-actor code):

```swift
extension BankAccount {
  func deposit(amount: Double) async {
    assert(amount >= 0)
    balance = balance + amount
  }
}
```

Now, the call to this method (which now must be adorned with [`await`](https://github.com/DougGregor/swift-evolution/blob/async-await/proposals/nnnn-async-await.md#await-expressions)) is well-formed:

```swift
await account.deposit(amount: amount)
```

Semantically, the call to `deposit(amount:)` is placed on the queue for the actor `account`, so that it will execute on that actor. If that actor is busy executing a task, then the caller will be suspended until the actor is available, so that other work can continue. See the section on [asynchronous calls](https://github.com/DougGregor/swift-evolution/blob/async-await/proposals/nnnn-async-await.md#asynchronous-calls) in the async/await proposal for more detail on the calling sequence.

> **Rationale**: by only allowing asynchronous instance methods of actor classes to be invoked from outside the actor, we ensure that all synchronous methods are already inside the actor when they are called. This eliminates the need for any queuing or synchronization within the synchronous code, making such code more efficient and simpler to write.

We can now properly implement a transfer of funds from one account to another:

```swift
extension BankAccount {
  func transfer(amount: Double, to other: BankAccount) async throws {
    assert(amount > 0)
    
    if amount > balance {
      throw BankError.insufficientFunds
    }

    print("Transferring \(amount) from \(ownerName) to \(other.ownerName)")

    // Safe: this operation is the only one that has access to the actor's local
    // state right now, and there have not been any suspension points between
    // the place where we checked for sufficient funds and here.
    balance = balance - amount
    
    // Safe: the deposit operation is queued on the `other` actor, at which 
    // point it will update the other account's balance.    
    await other.deposit(amount: amount)
  }
}
```

#### Closures and local functions

The restrictions on only allowing access to (non-`async`) actor-isolated declarations on `self` only work so long as we can ensure that the code in which `self` is valid is executing non-concurrently on the actor. For methods on the actor class, this is established by the rules described above: `async` function calls are serialized via the actor's queue, and non-`async` calls are only allowed when we know that we are already executing (non-concurrently) on the actor.

However, `self` can also be captured by closures and local functions. Should those closures and local functions have access to actor-isolated state on the captured `self`? Consider an example where we want to close out a bank account and distribute the balance amongst a set of accounts:

```swift
extension BankAccount {
  func close(distributingTo accounts: [BankAccount]) async {
    let transferAmount = balance / accounts.count

    accounts.forEach { account in 
      balance = balance - transferAmount             // is this safe?
      Task.runDetached {
        await account.deposit(amount: transferAmount)
      }  
    }
    
    thief.deposit(amount: balance)
  }
}
```

The closure is accessing (and modifying) `balance`, which is part of the `self` actor's isolated state. Once the closure is formed and passed off to a function (in this case, `Sequence.forEach`), we no longer have control over when and how the closure is executed. On the other hand, we "know" that `forEach` is a synchronous function that invokes the closure on successive elements in the sequence. It is not concurrent, and the code above would be safe.

If, on the other hand, we used a hypothetical parallel for-each, we would have a data race when the closure executes concurrently on different elements:

```swift
accounts.parallelForEach { account in 
  self.balance = self.balance - transferAmount    // DATA RACE!
  await account.deposit(amount: transferAmount)
}
```

In this proposal, a closure that is non-escaping is considered to be isolated within the actor, while a closure that is escaping is considered to be outside of the actor. This is based on a notion of when closures can be executed concurrently: to execute a particular closure on a different thread, one will have to escape the closure out of its current thread to run it on another thread. The rules that prevent a non-escaping closure from escaping therefore also prevent them from being executed concurrently. 

Based on the above, `parallelForEach` would need its closure parameter will be `@escaping`. The first example (with `forEach`) is well-formed, because the closure is actor-isolated and can access `self.balance`. The second example (with `parallelForEach`) will be rejected with an error:

```
error: actor-isolated property 'balance' is unsafe to reference in code that may execute concurrently
```

Note that the same restrictions apply to partial applications of non-`async` actor-isolated functions. Given a function like this:

```swift
extension BankAccount {
  func synchronous() { }
}
```

The expression `self.synchronous` is well-formed only if it is the direct argument to a function whose corresponding parameter is non-escaping. Otherwise, it is ill-formed because it could escape outside of the actor's context.

#### inout parameters

Actor-isolated stored properties can be passed into synchronous functions via `inout` parameters, but it is ill-formed to pass them to asynchronous functions via `inout` parameters. For example:

```swift
func modifiesSynchronously(_: inout Double) { }
func modifiesAsynchronously(_: inout Double) async { }

extension BankAccount {
  func wildcardBalance() async {
    modifiesSynchronously(&balance)        // okay
    await modifiesAsynchronously(&balance) // error: actor-isolated property 'balance' cannot be passed 'inout' to an asynchronous function
  }
}  
```

This restriction prevents exclusivity violations where the modification of the actor-isolated `balance` is initiated by passing it as `inout` to a call that is then suspended, and another task executed on the same actor then fails with an exclusivity violation in trying to access `balance` itself.

#### Escaping references

The rules concerning actor isolation ensure that accesses to an actor class's stored properties cannot occur concurrently, eliminating data races unless unsafe code has subverted the model. 

However, the actor isolation rules presented in this proposal are only sufficient for types which have *value semantics*. Under value semantics, any copy of the value produces a completely independent instance. Modifications to that independent instance cannot affect the original, and vice versa. Therefore, one can pass a copy of an actor-isolated stored property to another actor, or even write it into a global variable, and the actor will maintain its isolation because the copy is distinct.

*Reference semantics* break the isolation model, because mutations to a "copy" of a value which has reference semantics can affect the original, and vice versa. Let's introduce another stored property into our bank account to describe recent transactions, and make `Transaction` a type with reference semantics (a class with mutable state):

```swift
class Transaction { 
  var amount: Double
  var dateOccurred: Date
}

actor class BankAccount {
  // ...
  private var transactions: [Transaction]
}
```

The `transactions` stored property is actor-isolated, so it cannot be modified directly. Moreover, arrays have value semantics only when they contain types which also have value sematics. But the transactions stored in the array have reference semantics. The moment one of the instances of `Transaction`  from the `transactions` array *escapes* the actor's context, data isolation is lost. For example, here's a function that retrieves the most recent transaction:

```swift
extension BankAccount {
  func mostRecentTransaction() async -> Transaction? {   // UNSAFE! Transaction has reference semantics
    return transactions.min { $0.dateOccurred > $1.dateOccurred } 
  }
}
```

A client of this API gets a reference to the transaction inside the given bank account, e.g.,

```swift
guard let transaction = await account.mostRecentTransaction() else {
  return
}
```

At this point, the client can both modify the actor-isolated state by directly modifying the fields of `transaction`, as well as see any changes that the actor has made to the transaction. These operations may execute concurrently with code running on the actor, causing race conditions. 

Not all examples of "escaping" reference semantics are quite as straightforward as this one. Reference semantics can be imbued upon structs, enums, and in collections such as arrays and dictionaries (for example, by containing values of types which themselves have reference semantics), so one cannot look only at whether the type or its generic arguments are a `class`. The source of the reference semantics might also be hidden in code not visible to the user, e.g.,

```swift
public struct LooksLikeAValueType {
  private var transaction: Transaction  // does not have value semantics
}
```

Generics further complicate the matter: some types, like the standard library collections, have value semantics when their generic arguments do. An actor class might be generic, in which case its ability to maintain isolation depends on its generic argument:

```swift
actor class GenericActor<T> {
  private var array: [T]
  func first() async -> T? { 
    return array.first
  }
}
```

With this type, `GenericActor<Int>` maintains actor isolation but `GenericActor<Transaction>` does not.

There are solutions to these problems. However, the scope of the solutions is large enough that they deserve their own separate proposals. Therefore, **this proposal only provides basic actor isolation for data race safety for types with value semantics**.

### Global actors

What we’ve described as actor isolation is one part of a larger problem of data isolation.  It is important that all memory be protected from data races, not just memory directly associated with an instance of an actor class. Global actors allow code and state anywhere to be actor-isolated to a specific singleton actor. This extends the actor isolation rules out to annotated global variables, global functions, and members of any type or extension thereof. For example, global actors allow the important concepts of "Main Thread" or "UI Thread" to be expressed in terms of actors without having to capture everything into a single class. 

*Global actors* provide a way to annotate arbitrary declarations (properties, subscripts, functions, etc.) as being part of a process-wide singleton actor. A global actor is described by a type that has been annotated with the `@globalActor` attribute:

```swift
@globalActor
struct UIActor {
  /* details below */
}
```

Such types can then be used to annotate particular declarations that are isolated to the actor. For example, a handler for a touch event on a touchscreen device:

```swift
@UIActor
func touchesEnded(_ touches: Set<UITouch>, with event: UIEvent?) {
  // ...
}
```

A declaration with an attribute indicating a global actor type is actor-isolated to that global actor. The global actor type has its own queue that is used to perform any access to mutable state that is also actor-isolated with that same global actor.

Global actors are implicitly singletons, i.e. there is always _one_ instance of a global actor in a given process. This is in contrast to `actor classes`, of which there can be no instances, one instance, or many instances in a given process at any given time.


## Detailed design

### Actor classes

A class can be declared as an actor class using the `actor` modifier:

```
/// Declares a new type BankAccount
actor class BankAccount {
  // ...
}
```

Each instance of the actor class represents a unique actor.

An actor class may only inherit from another actor class. A non-actor class may only inherit from another non-actor class.

> **Rationale**: Actor classes enforce state isolation, but non-actor classes do not. If an actor class inherits from a non-actor class (or vice versa), part of the actor's state would not be covered by the actor-isolation rules, introducing the potential for data races on that state.

As a special exception described in the complementary proposal [Concurrency Interoperability with Objective-C](https://github.com/DougGregor/swift-evolution/blob/concurrency-objc/proposals/NNNN-concurrency-objc.md), an actor class may inherit from `NSObject`.

By default, the instance methods, properties, and subscripts of an actor class are actor-isolated to the actor instance. This is true even for methods added retroactively on an actor class via an extension, like any other Swift type.

```
extension BankAccount {
  func acceptTransfer(amount: Double) async { // actor-isolated
    balance += amount
  }
}  
```

An instance method, computed property, or subscript of an actor class may be annotated with `@actorIndependent` or a global actor attribute.  If so, it (or its accessors) are no longer actor-isolated to the `self` instance of the actor.

By default, the mutable stored properties (declared with `var`) of an actor class actor-isolated to the actor instance. A stored property may be annotated with `@actorIndependent(unsafe)` to remove this restriction. 

### Actor protocol

All actor classes conform to a new protocol `Actor`:

```swift
protocol Actor: AnyObject {
  func enqueue(partialTask: PartialAsyncTask)
}
```

The `enqueue(partialTask:)` operation is a low-level operation used to queue work for the actor to execute. `PartialAsyncTask` represents a unit of work to execute. It effectively has a single synchronous function, `run()`, which should be called synchronously within the actor's context. Only the compiler can produce new `PartialAsyncTasks`. To explicitly enqueue work on an actor, use the `run` method:

```swift
extension Actor {
  // Run the given async function on this actor.
  //
  // Precondition: the function is not constrained to a different actor;
  //   if it is not constrained to any actor at all, it will still run on
  //   behalf of `self`
  func run<T>(operation: () async throws -> T) async rethrows -> T
}
```

The `enqueue(partialTask:)` requirement is special in that it can only be provided in the primary actor class declaration (not an extension), and cannot be `final`. If `enqueue(partialTask:)` is not explicitly provided, the Swift compiler will provide a default implementation for the actor, with its own (hidden) queue.

> **Rationale**: This design strikes a balance between efficiency for the default actor implementation and extensibility to allow alternative actor implementations.   By forcing the method to be part of the main actor class, the compiler can ensure a common low-level implementation for actor classes that permits them to be passed as a single pointer and treated uniformly by the runtime.

Non-`actor` classes can conform to the `Actor` protocol, and are not subject to the restrictions above. This allows existing classes to work with some `Actor`-specific APIs, but does not bring any of the advantages of actor classes (e.g., actor isolation) to them.

### Global actors

A global actor can be declared by creating a new custom attribute type with `@globalActor`:

```swift
@globalActor
struct UIActor {
  static let shared = SomeActorInstance()
}
```

The type must provide a static `shared` property that provides the singleton actor instance, on which any work associated with the global actor will be enqueued. There are otherwise no requirements placed on the type itself.

The custom attribute type may be generic.  The custom attribute is called a global actor attribute.  A global actor attribute is never parameterized.  Two global actor attributes identify the same global actor if they identify the same type.

Global actor attributes apply to declarations as follows:

* A declaration cannot have multiple global actor attributes.  The rules below say that, in some cases, a global actor attribute is propagated from one declaration to another.  If the rules say that an attribute “propagates by default”, then no propagation is performed if the destination declaration has an explicit global actor attribute.  If the rules say that attribute “propagates mandatorily”, then it is an error if the destination declaration has an explicit global actor attribute that does not identify the same actor.  Regardless, it is an error if global actor attributes that do not identify the same actor are propagated to the same declaration.

* A function, property, subscript, or initializer declared with a global actor attribute becomes actor-isolated to the given global actor.

 ```swift
 @UIActor func drawAHouse(graphics: CGGraphics) {
     // ...
 }
 ```

* Local variables and constants cannot be marked with a global actor attribute. 

* A type declared with a global actor attribute propagates the attribute to all methods, properties, subscripts, and extensions of the type by default.

* An extension declared with a global actor attribute propagates the attribute to all the members of the extension by default.

* A protocol declared with a global actor attribute propagates the attribute to its conforming types by default.

* A protocol requirement declared with a global actor attribute propagates the attribute to its witnesses mandatorily if they are declared in the same module as the conformance. 

* A class declared with a global actor attribute propagates the attribute to its subclasses mandatorily.

* An overridden declaration propagates its global actor attribute (if any) to its overrides mandatorily.  Other forms of propagation do not apply to overrides.  It is an error if a declaration with a global actor attribute overrides a declaration without an attribute.

* An actor class cannot have a global actor attribute.  Stored instance properties of actor classes cannot have global actor attributes.  Other members of an actor class can have global actor attributes; such members are actor-isolated to the global actor, not the actor instance.

* A deinit cannot have a global actor attribute and is never a target for propagation.

The effect of these rules is to make it easy for a few classes and protocols to be annotated as being part of a global actor (e.g., the `@UIActor`), and for code that interoperates with those (subclassing the classes, conforming to the protocols) to not need explicit annotations.

### Actor-independent declarations

A declaration may be declared to be actor-independent:

```
@actorIndependent
var count: Int { constantCount + 1 }
```

When used on a declaration, it indicates that the declaration is not actor-isolated to any actor, which allows it to be accessed from anywhere. Moreover, it interrupts the implicit propagation of actor isolation from context, e.g., it can be used on an instance declaration in an actor class to make the declaration actor-independent rather than isolated to the actor.

When used on a class, the attribute applies by default to members of the class and extensions thereof.  It also interrupts the ordinary implicit propagation of actor-isolation attributes from the superclass, except as required for overrides.

The attribute is ill-formed when applied to any other declaration.  It is ill-formed if combined with an explicit global actor attribute.

The `@actorIndependent` attribute has an optional "unsafe" argument.  `@actorIndependent(unsafe)` is treated the same way as `@actorIndependent` from the client's perspective, meaning that it can be used from anywhere. However, the implementation of an `@actorIndependent(unsafe)` entity is allowed to refer to actor-isolated state, which would have been ill-formed under `@actorIndependent`.

### Actor isolation checking

Any given non-local declaration in a program can be classified into one of five actor isolation categories:

* Actor-isolated to a specific instance of an actor class:
  - This includes the stored instance properties of an actor class as well as computed instance properties, instance methods, and instance subscripts, as demonstrated with the `BankAccount` example.
* Actor-isolated to a specific global actor:
  - This includes any property, function, method, subscript, or initializer that has an attribute referencing a global actor, such as the `touchesEnded(_:with:)` method mentioned above.
* Actor-independent: 
  - The declaration is not actor-isolated to any actor. This includes any property, function, method, subscript, or initializer that has the `@actorIndependent` attribute.
* Actor-independent (unsafe): 
  - The declaration is not actor-isolated to any actor. This includes any property, function, method, subscript, or initializer that has the `@actorIndependent(unsafe)` attribute.
  - The declaration's definition is not subject to actor isolation checking.
* Unknown: 
  - The declaration is not actor-isolated to any actor, nor has it been explicitly determined that it is actor-independent. Such code might depend on shared mutable state that hasn't been modeled by any actor.

The actor isolation rules are checked in a number of places, where two different declarations need to be compared to determine if their usage together maintains actor isolation. There are several such places:
* When the definition of one declaration (e.g., the body of a function) accesses another declaration in executuable code, e.g., calling a function, accessing a property, or evaluating a subscript.
* When one declaration overrides another.
* When one declaration satisfies a protocol requirement.

We'll describe each scenario in detail.

#### Accesses in executable code

A given declaration (call it the "source") can access another declaration (call it the "target") in executable code, e.g., by calling a function or accessing a property or subscript. If the target is `async`, there is nothing more to check: the call will be scheduled on the target actor's queue, so the access is well-formed.

When the target is not `async`, the actor isolation categories for the source and target must be compatible. A source and target category pair is compatible if:
* the source and target categories are the same,
* the target category is actor-independent or actor-independent (unsafe),
* the source category is actor-independent (unsafe), or
* the target category is unknown.

The first rule is the most direct: an actor-isolated declaration can access other declarations within its same actor, whether that's an actor instance (on `self`) or global actor (e.g., `@UIActor`).

The second rule specifies that actor-independent declarations can be used from anywhere because they aren't tied to a particular actor. Actor classes can provide actor-independent instance methods, but because those functions are not actor-isolated, that cannot read the actor's own mutable state. For example:

```swift
extension BankAccount {
  @actorIndependent
  func greeting() -> String {
    return "Hello, \(ownerName)!"  // okay: ownerName is immutable
  }
  
  @actorIndependent
  func steal(amount: Double) {
    balance -= amount  // error: actor-isolated property 'balance' can not be referenced from an '@actorIndependent' context
  }
}  
```

The third rule is an unsafe opt-out that allows a declaration to be treated as actor-independent by its clients, but can do actor-isolation-unsafe operations internally. It is intended to be used sparingly for interoperability with existing  synchronization mechanisms or low-level performance tuning.

```swift
extension BankAccount {
  @actorIndependent(unsafe)
  func steal(amount: Double) {
    balance -= amount  // data-racy, but permitted due to (unsafe)
  }
}
```

The fourth rule is provided to allow interoperability between actors and existing Swift code. Actor code (which by definition is all new code) can call into existing Swift code with unknown actor isolation. However, code with unknown actor isolation cannot call back into (non-`async`) actor-isolated code, because doing so would violate the isolation guarantees of that actor. This allows incremental adoption of actors into existing code bases, isolating the new actor code while allowing them to interoperate with the rest of the code.

#### Overrides

When a given declaration (the "overriding declaration") overrides another declaration (the "overridden" declaration), the actor isolation of the two declarations is compared. The override is well-formed if:

* the overriding and overridden declarations have the same actor isolation or
* the overriding declaration is actor-independent.

In the absence of an explicitly-specified actor-isolation attribute (i.e, a global actor attribute or `@actorIndependent`), the overriding declaration will inherit the actor isolation of the overridden declaration.

#### Protocol conformance

When a given declaration (the "witness") satisfies a protocol requirement (the "requirement"), the actor isolation of the two declarations is compared. The protocol requirement can be satisfied by the witness if:

* the witness and requirement have the same actor isolation,
* the witness and requirement are `async` and the requirement has unknown actor isolation, or
* the witness is actor-independent and the requirement has unknown actor isolation.

The last case is particularly important to allow actor classes to conform to existing protocols, which will have synchronous requirements. For example, say that we want to make our `BankAccount` actor class conform to `CustomStringConvertible`:

```swift
extension BankAccount: CustomStringConvertible {
  var description: String {       // error: actor-isolated property "description" cannot be used to satisfy a protocol requirement
    "Bank account of \"\(ownerName)\""
  }
}
```

One can use `@actorIndependent` on such declarations to allow them to satisfy synchronous protocol requirements:

```swift
extension BankAccount: CustomStringConvertible {
  @actorIndependent
  var description: String {
    "Bank account of \"\(ownerName)\""
  }
}
```

In the absence of an explicitly-specified actor-isolation attribute, a witness that is defined in the same type or extension as the conformance for the requirement's protocol will have its actor isolation inferred from the protocol requirement.

## Source compatibility

This proposal is additive, and should not break source compatibility. The addition of the `actor` contextual keyword to introduce actor classes is a parser change that does not break existing code, and the other changes are carefully staged so they do not change existing code. Only new code that introduces actor classes or actor-isolation attributes will be affected.

## Effect on ABI stability

This is purely additive to the ABI.

## Effect on API resilience

Nearly all changes in actor isolation are breaking changes, because the actor isolation rules require consistency between a declaration and its users:

* A class cannot be turned into an actor class or vice versa.
* The actor isolation of a public declaration cannot be changed except between `@actorIndependent(unsafe)` and `@actorIndependent`.


