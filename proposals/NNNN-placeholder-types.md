# Placeholder types

* Proposal: [SE-NNNN](NNNN-placeholder-types.md)
* Authors: [Frederick Kellison-Linn](https://github.com/jumhyn)
* Review Manager: TBD
* Status: **Awaiting implementation**

## Introduction

When Swift's type inference is unable to work out the type of a particular expression, it requires the programmer to provide the necessary type context explicitly. However, all mechanisms for doing this require the user to write out the entire type signature, even if only one portion of that type is actually needed by the compiler. E.g.,

```swift
func foo(_ x: String) -> Double { return Double(x.count) / 2.0 }
func foo(_ x: Int) -> Double { return Double(x) }

let stringTransform = foo as (String) -> Int
```
In the above example, we only really need to clarify the *argument* type—once that's determined, the return type is a given. This proposal allows the user to provide type hints which use *placeholder types* in such circumstances, so that the initialization of `stringTransform` could be written as:

```swift
let stringTransform = foo as (String) -> _
```

Swift-evolution thread: [Partial type annotations](https://forums.swift.org/t/partial-type-annotations/41239)

## Motivation

Swift's type inference system is quite powerful, but there are many situations where it is impossible (or simply infeasible) for the compiler to work out the type of an expression, or where the user needs to override the default types worked out by the compiler. The example above of an overloaded function `foo` is one such example.

Fortunately, Swift provides several ways for the user to provide type information explicitly. Common forms are:
- Variable type annotations:
```swift
let stringTransform: (String) -> Double = foo
```

- Type coercion via `as` (seen above):
```swift
let stringTransform = foo as (String) -> Double
```

- Passing type parameters explicitly (e.g., `JSONDecoder`)
```swift
let dict = JSONDecoder().decode([String: Int].self, from: data)
```

The downside of all of these approaches is that they require the user to write out the *entire* type, even when the compiler only needs guidance on some sub-component of that type. This can become particularly problematic in cases where a complex type that would normally be inferred has to be written out explicitly because some *unrelated* portion of the type signature is required. E.g.,

```swift
enum Either<Left, Right> {
  case left(Left)
  case right(Right)
  
  init(left: Left) { self = .left(left) }
  init(right: Right) { self = .right(right) }
}

func makePublisher() -> Some<Complex<Nested<Publisher<Chain<Int>>>>> { ... }
```
Attempting to initialize an `Either` from `makePublisher` isn't as easy as one might like:
```swift
let publisherOrValue = Either(left: makePublisher()) // Error: generic parameter 'Right' could not be inferred
```

Instead, we have to write out the full generic type:

```swift
let publisherOrValue = Either<Some<Complex<Nested<Publisher<Chain<Int>>>>>, Int>(left: makePublisher())
```

The resulting expression is more difficult to write *and* read. If `Left` were the result of a long chain of Combine operators, the author may not even know the correct type to write and would have to glean it from several pages of documentation or compiler error messages.

## Proposed solution

Allow users to write types with designated *placeholder types* (spelled "`_`") which indicate that the corresponding type should be filled in during type checking. For the above `publisherOrValue` example, this would look like:

```swift
let publisherOrValue = Either<_, Int>(left: makePublisher())
```

Because the generic argument to the `Left` parameter can be inferred from the return type of `makePublisher`, we do not need to write it out. Instead, during type checking, the compiler will see that the first generic argument to `Either` is a placeholder and leave it unresolved until other type information can be used to fill it in.

## Detailed design

### Grammar

This proposal introduces the concept of a user-specified "placeholder type," which, in terms of the grammar, can be written anywhere a type can be written. In particular, the following productions will be introduced:

```
type → placeholder-type
placeholder-type → '_'
```

Examples of types containing placeholders are:

```swift
Array<_> // array with placeholder element type
[Int: _] // dictionary with placeholder value type
(_) -> Int // function type accepting a single placeholder type argument and returning 'Int'
@escaping _ // attributed placeholder type
(_, Double) // tuple type of placeholder and 'Double'
_? // optional wrapping a placeholder type
_ // a bare placeholder type is also ok!
```

### Type inference

When the type checker encounters a type containing a placeholder type, it will fill in all of the non-placeholder context exactly as before. Placeholder types will be treated as providing no context for that portion of the type, requiring the rest of the expression to be solvable given the partial context. Effectively, placeholder types act as user-specified anonymous type variables that the type checker will attempt to solve using other contextual information.

Let's examine a concrete example:

```swift
import Combine

func makeValue() -> String { "" }
func makeValue() -> Int { 0 }

let publisher = Just(makeValue()).setFailureType(to: Error.self).eraseToAnyPublisher()
```

As written, this code is invalid. The compiler complains about the "ambiguous use of `makeValue()`" because it is unable to determine which `makeValue` overload should be called. We could solve this by providing a full type annotation:

```swift
let publisher: AnyPublisher<Int, Error> = Just(makeValue()).setFailureType(to: Error.self).eraseToAnyPublisher()
```

Really, though, this is overkill. The generic argument to `AnyPublisher`'s `Failure` parameter is clearly `Error`, since the result of `setFailureType(to:)` has no ambiguity. Thus, we can substitute in a placeholder type for the `Failure` parameter, and still successfully typecheck this expression:

```swift

let publisher: AnyPublisher<Int, _> = Just(makeValue()).setFailureType(to: Error.self).eraseToAnyPublisher()
```

Now, the type checker has all the information it needs to resolve the reference to `makeValue`: the ultimately resulting `AnyPublisher` must have `Output == Int`, so the result of `setFailureType(to:)` must have `Output == Int`, so the instance of `Just` must have `Output == Int`, so the argument to `Just.init` must have type `Int`, so `makeValue` must refer to the `Int`-returning overload!

Note: it's technically legal to specify a bare placeholder type, which may act as a more explicit way to call out "this type is inferred":

```swift
let percent: _ = 100.0
```

### Generic constraints

In some cases, placeholders may be expected to conform to certain protocols. E.g., it is perfectly legal to write:

```swift
let dict: [_: String] = [0: "zero", 1: "one", 2: "two"]
```

When examining the storage type for `dict`, the compiler will expect the placeholder key type to conform to `Hashable`. Conservatively, placeholder types are assumed to satisfy all necessary constraints, deferring the verification of these constraints until the checking of the intialization expression.

### Generic parameter inference

A limited version of this feature is already present in the language via generic parameter inference. When the generic arguments to a generic type can be inferred from context, you are permitted to omit them, like so:

```swift
import Combine

let publisher = Just(0) // Just<Int> is inferred!
```

With placeholder types, writing the bare name of a generic type (in most cases, see note below) becomes equivalent to writing the generic signature with placeholder types for the generic arguments. E.g., the initialization of `publisher` above is the same as:

```swift
let publisher = Just<_>(0)
```

Note: there is an existing rule that *inside the body* of a generic type `S<T1, ..., Tn>`, the bare name `S` is equivalent to `S<T1, ..., Tn>`. This proposal does not augment this rule nor attempt to express this rule in terms of placeholder types.

### Function signatures

As is the case today, function signatures under this proposal are required to have their argument and return types fully specified. Generic parameters cannot be inferred and placeholder types are not permitted to appear within the signature, even if the type could ostensibly be inferred from e.g., a protocol requirement or default argument expression. 

Thus, it is an error under this proposal to write something like:

```swift
func doSomething(_ count: _? = 0) { ... }
```

just as it would be an error to write:

```swift
func doSomething(_ count: Optional = 0) { ... }
```

even though the type checker could infer the `Wrapped` type in an expression like:

```swift
let count: _? = 0
```

## Source compatibility

This is an additive change with no effect on source compatibility.

## Effect on ABI stability

This feature does not have any effect on the ABI.

## Effect on API resilience

Placeholder types are not exposed as API. In a compiled interface, they are replaced by whatever type the type checker fills in for the placeholder. While the introduction or removal of a placeholder *on its own* is not necessarily an API or ABI break, authors should be careful that the introduction/removal of the additional type context does not ultimately change the inferred type of the variable.

## Alternatives considered

### Alternative spellings

Several potential spellings of the placeholder type were suggested, with most users preferring either "`_`" or "`?`". The question mark version was rejected primarily for the fact that the existing usage of `?` in the type grammar for optionals would be confusing and or ambiguous if it were overloaded to also stand for a placeholder type.

Some users also worried that the underscore spelling would preclude the same spelling from being used for automatically type-erased containers, e.g.,

```swift
var anyArray: Array<_> = [0]
anyArray = ["string"]
let stringArray = anyArray as? Array<String>
```

This objection to the `_` is compelling, but it was noted during discussion that usage of an explicit existential marker keyword (a la `any Array<_>`) could allow the usage of an underscore for both placeholder types and erased types.

At the pitch phase, the author remains open to alternative spellings for this feature. In particular, the "`any Array<_>`" resolution does not address circumstances where an author may want to both erase some components of a type but allow inference to fill in others.

## Future directions

### Placeholders for generic bases

In some examples, we're still technically providing more information than the compiler strictly needs to determine the type of an expression. E.g., in the example from the **Type inference** section, we could have conceivably written the type annotation as:

```swift
let publisher: _<Int, _> = Just(makeValue()).setFailureType(to: Error.self).eraseToAnyPublisher()
```

Since the type of the generic `AnyPublisher` base is fully determined from the result type of `eraseToAnyPublisher()`. The author is skeptical that this ultimately results in clearer code, and so opts to defer consideration of such a feature until there is further consideration.
