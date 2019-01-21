# Opaque Result Types for Public Interfaces

* Proposal: [SE-NNNN](NNNN-opaque-result-types.md)
* Author: [Doug Gregor](https://github.com/DougGregor)
* Review Manager: TBD
* Status: **Awaiting review**
* Pull Request: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN)


## Table of Contents
* [Introduction](#introduction)
* [Proposed solution](#proposed-solution)
  + [Opaque result types vs. existentials](#opaque-result-types-vs-existentials)
  + [Type identity](#type-identity)
  + [Implementing a function returning an opaque type](#implementing-a-function-returning-an-opaque-type)
  + [Properties and subscripts](#properties-and-subscripts)
  + ["Naming" the opaque result type](#-naming--the-opaque-result-type)
  + [Conditional conformance](#conditional-conformance)
* [Detailed design](#detailed-design)
  + [Grammar of opaque result types](#grammar-of-opaque-result-types)
  + [Restrictions on opaque result types](#restrictions-on-opaque-result-types)
  + [Single opaque result type per entity](#single-opaque-result-type-per-entity)
  + [Uniqueness of opaque result types](#uniqueness-of-opaque-result-types)
  + [Ambiguity with `where` clauses](#ambiguity-with--where--clauses)
  + [Implementation strategy](#implementation-strategy)
* [Source compatibility](#source-compatibility)
* [Effect on ABI stability](#effect-on-abi-stability)
* [Effect on API resilience](#effect-on-api-resilience)
* [Rust's `impl Trait`](#rust-s--impl-trait-)
* [Future Directions](#future-directions)
  + [Opaque type aliases](#opaque-type-aliases)


## Introduction

This proposal introduces the ability to "hide" the specific result type of a function from its callers. The result type is described only by its capabilities, e.g., a `Collection` with a specific `Element` type. Clients can use the resulting values freely, but the underlying concrete type is known only to the implementation of the function and need not be spelled out explicitly.

Consider an operation like the following:

```swift
let resultValues = values.lazy.map(transform).filter { $0 != nil }.map { $0! }
```

If we want to encapsulate that code in another method, we're going to have to write out the type of that lazy-map-filter chain, which is rather ugly:

```swift
extension LazyMapCollection {
  public func compactMap<ElementOfResult>(
                _ transform: @escaping (Elements.Element) -> ElementOfResult?
              )
      -> LazyMapSequence<
           LazyFilterSequence<
             LazyMapSequence<Elements, ElementOfResult?>
           >,
           ElementOfResult
         > {
    return self.map(transform).filter { $0 != nil }.map { $0! }
  }
}
```

The author of `compactMap(_:)` doesn't want to have to figure out that type,
and the clients of `compactMap(_:)` don't want to have to reason about it. `compactMap(_:)` is the subject of [SE-0222](https://github.com/apple/swift-evolution/blob/master/proposals/0222-lazy-compactmap-sequence.md), which introduces a new standard library type `LazyCompactMapCollection` for the sole purpose of describing the result type of `compactMap(_:)`:

```swift
extension LazyMapCollection {
  public func compactMap<ElementOfResult>(
                _ transform: @escaping (Element) -> ElementOfResult?
              )
      -> LazyCompactMapCollection<Base, ElementOfResult> {
    return LazyCompactMapCollection<Base, ElementOfResult>(/*...*/)
  }
}
```

This is less verbose, but requires the introduction of a new, public type solely to describe the result of this function, increasing the surface area of the standard library and requiring a nontrivial amount of implementation.

Opaque result types allow the function to state the capabilities of its result type without tying it down to a concrete type. For example, `compactMap(_:)` would state that its result type is "an opaque `Collection` whose `Element` type is `ElementOfResult `. The implementation could return the original method chain:

```swift
extension LazyMapCollection {
  public func compactMap<ElementOfResult>(
                _ transform: @escaping (Element) -> ElementOfResult?
              )
      -> opaque Collection where _.Element == ElementOfResult {
    return self.map(transform).filter { $0 != nil }.map { $0! }
  }
}
```

or a custom *potentially `private`* type:

```swift
private struct LazyCompactMapCollection<Base: Collection, Element> { ... }

extension LazyMapCollection {
  public func compactMap<ElementOfResult>(
                _ transform: @escaping (Element) -> ElementOfResult?
              )
      -> opaque Collection where _.Element == ElementOfResult {
    return LazyCompactMapCollection<Base, ElementOfResult>(/*...*/)
  }
}
```

Either way, clients only know that they are getting a `Collection` whose `Element` type is `ElementOfResult`. The underlying concrete type is hidden, and can even change from one version of the library to the next without breaking those clients, because the actual type identity was never exposed. This allows us to provide potentially-more-efficient implementations without expanding the surface area of the library.

Swift-evolution thread: [Opaque result types](https://forums.swift.org/t/opaque-result-types/15645)

## Proposed solution

This proposal introduces an `opaque` type that can be used to describe an opaque result type for a function. `opaque` types can only be used in the result type of a function, the type of a property, or the element type of a subscript declaration. An `opaque` type is backed by a specific concrete type, but that type is only known to the implementation of that function/property/subscript. Everywhere else, the type is opaque, and is described by its characteristics and originating function/property/subscript. For example:

```swift
func makeMeACollection<T>(_: T.Type)
     -> opaque MutableCollection & RangeReplaceableCollection where _.Element == T {
   return [T]()   // okay: an array of T satisfies all of the requirements
}
```

The syntax of the "opaque" type is derived from the discussions of [generalized existentials](https://github.com/austinzheng/swift-evolution/blob/az-existentials/proposals/XXXX-enhanced-existentials.md). Essentially, following the `opaque` keyword is a class, protocol, `AnyObject`, or composition thereof (joined with `&`), optionally followed by a `where` clause indicating other requirements. Within that `where` clause, one can use `_` to refer to the opaque result type itself, so `_.Element` would refer to the `Element` associated type of the `opaque` type (which otherwise cannot be named).

A caller to `makeMeACollection(_:)` can rely on the result type satisfying all of the requirements listed. For example:

```swift
var c = makeMeACollection(Int.self)
c.append(17)         // okay: it's a RangeReplaceableCollection with Element == Int
c[c.startIndex] = 42 // okay: it's a MutableCollection with Element == Int
print(c.reversed())  // okay: all Collection/Sequence operations are available

func foo<C: Collection>(_ : C) { }
foo(c)               // okay: C inferered to opaque result type of makeMeACollection()
```

Moreover, opaque result types to be used freely with other generics, e.g., forming a collection of the results:

```swift
var cc = makeMeACollection(type(of: c))
cc.append(c)         // okay: Element == the result type of makeMeACollection
var c2 = makeMeACollection(Int.self)
cc.append(c2)        // okay: Element == the result type of makeMeACollection
```

### Opaque result types vs. existentials
On the surface, opaque result types are quite similar to (generalized) existential types: in each case, the specific concrete type is unknown to the static type system, and can be manipulated only through the stated capabilities (e.g., protocol and superclass constraints). There are some similarities between the two features for code where the identity of the return type does not matter. For example:

```swift
protocol P
  func foo()
}

func existentialP() -> P { /* ... */ }
func opaqueP() -> opaque P { /* ... */ }

existentialP().foo()   // okay
opaqueP().foo()   // okay
```

However, the fundamental difference between opaque result types and existentials revolves around type identity. All instances of an opaque result type are guaranteed to have the same type at run time, whereas different instances of an existential type may have different types at run time. It is this aspect of existential types that makes their use so limited in Swift. For example, consider a function that takes two values of (existential) type `Equatable` and tries to compare them:

```swift
protocol Equatable {
  static func ==(lhs: Self, rhs: Self) -> Bool
}

func isEqual(_ x: Equatable, y: Equatable) -> Bool {
  return x == y
}
```

The `==` operator is meant to take two values of the same type and compare them. It's clear how that could work for a call like `isEqual(1, 2)`, because both `x` and `y` store values of type `Int`.

But what about a call `isEqual(1, "one")`? Both `Int` and `String` are `Equatable`, so the call to `isEqual` should be well-formed. However, how would the evaluation of `==` work? There is no operator `==` that works with an `Int` on the left and a `String` on the right, so it would fail at run-time with a type mismatch.

Swift rejects the example with the following diagnostic:

```
error: protocol 'Equatable' can only be used as a generic constraint because it has Self or associated type requirements
```

The generalized existentials proposal dedicates quite a bit of its design space to [ways to check whether two instances of existential type contain the same type at runtime](https://github.com/austinzheng/swift-evolution/blob/az-existentials/proposals/XXXX-enhanced-existentials.md#real-types-to-existentials-associated-types) to cope with this aspect of existential types. Generalized existentials can make it *possible* to cope with values of `Equatable` type, but it can't make it easy. The following is a correct implementation of `isEqual` using generalized existentials:

```swift
func isEqual(_ x: Equatable, y: Equatable) -> Bool {
  if let yAsX = y as? x.Self {
    return x == yAsX
  }
  
  if let xAsY = x as? y.Self {
    return xAsY == y
  }
  
  return false
}
```

Note that the user must explicitly cope with the potential for run-time type mismatches, because the Swift language will not implicitly defer type checking to run time.

Existentials also interact poorly with generics, because a value of existential type does not conform to its own protocol. For example:

```swift
protocol P { }

func acceptP<T: P>(_: T) { }
func provideP(_ p: P) {
  acceptP(p) //  error: protocol type 'P' cannot conform to 'P' because only 
             // concrete types can conform to protocols
}
```

[Hamish](https://stackoverflow.com/users/2976878/hamish) provides a [complete explanation on StackOverflow](https://stackoverflow.com/questions/33112559/protocol-doesnt-conform-to-itself) as to why an existential of type `P` does not conform to the protocol `P`. The following example from that answer demonstrates the point with an initializer requirement:

```swift
protocol P {
  init()
}

struct S: P {}
struct S1: P {}

extension Array where Element: P {
  mutating func appendNew() {
    // If Element is P, we cannot possibly construct a new instance of it, as you cannot
    // construct an instance of a protocol.
    append(Element())
  }
}

var arr: [P] = [S(), S1()]

// error: Using 'P' as a concrete type conforming to protocol 'P' is not supported
arr.appendNew()
```

Hamish notes that:

> We cannot possibly call `appendNew()` on a `[P]`, because `P` (the `Element`) is not a concrete type and therefore cannot be instantiated. It must be called on an array with concrete-typed elements, where that type conforms to `P`.

The major limitations that Swift places on existentials are, fundamentally, because different instances of existential type may have different types at run time. Generalized existentials can lift some restrictions (e.g., they can allow values of type `Equatable` or `Collection` to exist), but they cannot make the potential for run-time type conflicts disappear without weakening the type-safety guarantees provided by the language (e.g., `x == y` for `Equatable` `x` and `y` will still be an error) nor make existentials as powerful as concrete types (existentials still won't conform to their own protocols).

Opaque result types have none of these limitations, because an opaque result type is a name for a fixed-but-hidden concrete type. If a function returns an `opaque Equatable` result type, one can compare the results of successive calls to the function with `==`:

```swift
func getEquatable() -> opaque Equatable {
  return Int.random(in: 0..<10)
}

let x = getEquatable()
let y = getEquatable()
if x == y {           // okay: calls to getEquatable() always return values of the same type
  print("Bingo!")
}
```

Opaque result types *do* conform to the protocols they name, because opaque result types are another name for a concrete type that is guaranteed to conform to those protocols. For example:

```swift
func isEqualGeneric<T: Equatable>(_ lhs: T, _ rhs: T) -> Bool {
  return lhs == rhs
}

let x = getEquatable()
let y = getEquatable()
if isEqual(x, y) {           // okay: the opaque result of getEquatable() conforms to Equatable
  print("Bingo!")
}
```

(Generalized) existentials are well-suited for use in heterogeneous collections, or other places where one expects the run-time types of values to vary and there is little need to compare two different values of existential type. However, they don't fit the use cases outlined for opaque result types, which require the types that result from calls to compose well with generics and provide the same capabilities as a concrete type.

### Type identity
An opaque result type is not considered equivalent to its underlying type by the static type system:

```swift
var intArray = [Int]()
cc.append(intArray)         // error: [Int] is not known to equal the result type of makeMeACollection
```

However, as with generic type parameters, one can inspect an opaque type's underlying type at runtime. For example, a conditional cast could determine whether the result of `makeMeACollection` is of a particular type:

```swift
if let arr = makeMeACollection(Int.self) as? [Int] {
  print("It's an [Int], \(arr)\n")
} else {
  print("Guessed wrong")
}
```

In other words, opaque result types are only opaque to the static type system: they don't exist at runtime.

### Implementing a function returning an opaque type 
The implementation of a function returning an opaque type must return a value of the same concrete type `T` from each `return` statement, and `T` must meet all of the constraints stated on the opaque type. For example:

```swift
protocol P { }
extension Int : P { }
extension String : P { }

func f1() -> opaque P {
  return "opaque"
}

func f2(i: Int) -> opaque P {   // okay: both returns produce Int
  if i > 10 { return i }
  return 0
}

func f2(flip: Bool) -> opaque P {
  if flip { return 17 }
  return "a string"       // error: different return types Int and String
}

func f3() -> opaque P {
  return 3.1419           // error: Double does not conform to P
}

func f4() -> opaque P {
  let p: P = "hello"
  return p                // error: protocol type P does not conform to P
}

func f5() -> opaque P {
  return f1()             // okay: f1() returns an opaque type that conforms to P
}

protocol Initializable { init() }

func f6<T: P & Initializable>(_: T.Type) -> opaque P {
  return T()              // okay: T will always be a concrete type conforming to P
}
```

These rules guarantee that there is a single concrete type produced by any call to the function. The concrete type can depend on the generic type arguments (as in the `f6()` example), but must be consistent across all `return` statements.

Note that recursive calls *are* allowed, and are known to produce a value of the same concrete type, but that the concrete type itself is not known:

```swift
func f7(_ i: Int) -> opaque P {
  if i == 0 {
    return f7(1)                 // okay: returning the opaque result type of f(), similar to f5()
  } else if i < 0 {
    let result: Int = f7(-i)     // error: opaque result type of f() is not convertible to Int
    return result
  } else {
    return 0
  }
}
```

Of course, there must be at least one `return` statement that provides a concrete type!

### Properties and subscripts

Opaque result types can be used with properties and subscripts. For example, the `lazy` property of a collection could have an opaque result type:

```swift
extension Collection {
  var lazy: opaque Collection where _.Element == Element { ... }
}
```

For computed properties, the concrete type is determined by the `return` statements in the getter. Opaque result types can also be used in stored properties that have an initializer, in which case the concrete type is the type of the initializer:

```swift
let strings: opaque Collection where _.Element == String = ["hello", "world"]
```

Properties and subscripts of opaque result type can be mutable. For example:

```swift
// Module A
public protocol P {
  mutating func flip()
}

private struct Witness: P {
  mutating func flip() { ... }
}

public var someP: opaque P = Witness()

// Module B
import A
someP.flip()  // okay: flip is a mutating function called on a variable
```

With a subscript or a computed property, the type of the value provided to the setting (e.g., `newValue`) is determined by the `return` statements in the getter, so the type is consistent and known only to the implementation of the property or subscript. For example:

```swift
protocol P { }
private struct Impl: P { }

public struct Vendor {
  private var storage: [Impl] = [...]

  public var count: Int {
    return storage.count
  }

  public subscript(index: Int) -> opaque P {
    get {
      return storage[index]
    }
    set (newValue) {
      storage[index] = newValue
    }
  }
}

var vendor = Vendor()
vendor[0] = vendor[2] // okay: can move elements around
```

### "Naming" the opaque result type
While one can use type inference to declare variables of the opaque result type of a function, there is no direct way to name the opaque result type:

```swift
func f1() -> opaque P { /* ... */ }

let vf1 = f1()    // type of vf1 is the opaque result type of f1()
```

However, the type inference used to satisfy associated type requirements can be used to give a name to the opaque result type. For example:

```swift
protocol P { }

protocol Q {
  associatedtype SomeType: P
  
  func someValue() -> SomeType
}

struct S: Q {
  func someValue() -> opaque P {
    /* ... */
  }
  
  /* infers typealias SomeType = opaque result type of S.someValue() */
}

let sv: S.SomeType     // okay: names the opaque result type of S.someValue()
sv = S().someValue()   // okay: returns the same opaque result type
```

Note that having a name for the opaque result type still doesn't give information about the underlying concrete type. For example, the only way to create an instance of the type `S.SomeType` is by calling `S.someValue()`.

### Conditional conformance

When a generic function returns an adapter type, it's not uncommon for the adapter to use [conditional conformances](https://github.com/apple/swift-evolution/blob/master/proposals/0143-conditional-conformances.md) to reflect the capabilities of its underlying type parameters. For example, consider the `reversed()` operation:

```swift
extension BidirectionalCollection {
  public func reversed() -> ReversedCollection<Self> {
    return ReversedCollection<Self>(...)
  }
}
```

`ReversedCollection` is always a `BidirectionalCollection`, with a conditional conformance to `RandomAccessCollection`:

```swift
public struct ReversedCollection<C: BidirectionalCollection>: BidirectionalCollection {
  /* ... */
}

extension ReversedCollection: RandomAccessCollection
    where C: RandomAccessCollection {
  /* ... */
}
```

What happens if we hid the `ReversedCollection` adapter type behind an opaque result type?

```swift
extension BidirectionalCollection {
  public func reversed() -> opaque BidirectionalCollection
      where _.Element == Element {
    return ReversedCollection<Self>(...)
  }
}
```

Now, clients that call `reversed()` on a `RandomAccessCollection` (like an array), would get back something that is only known to be a `BidirectionalCollection`: there would be no way to treat it as a `RandomAccessCollection`. The library could provide another overload of `reversed()`:

```swift
extension RandomAccessCollection {
  public func reversed() -> opaque RandomAccessCollection
      where _.Element == Element {
    return ReversedCollection<Self>(...)
  }
}
```

However, doing so is messy, and the client would have no way to know that the type returned by the two `reversed()` functions are, in fact, the same. To express the conditional conformance behavior, we extend the syntax of opaque result types to describe additional capabilities of the resulting type that depend on extended requirements. For example, we could state that the result of `reversed()` is *also* a `RandomAccessCollection` when `Self` is a `RandomAccessCollection`. One possible syntax:

```swift
extension BidirectionalCollection {
  public func reversed() 
      -> opaque BidirectionalCollection where _.Element == Element
         opaque RandomAccessCollection where Self: RandomAccessCollection {
    return ReversedCollection<Self>(...)
  }
}
```

Here, we add a second `opaque` clause that states additional information about the opaque result type (it is a `RandomAccessCollection`) as well as the requirements under which that capability is available (the `where Self: RandomAccessCollection`). One could have multiple conditional clauses, e.g.,

```swift
extension BidirectionalCollection {
  public func reversed() 
      -> opaque BidirectionalCollection where _.Element == Element
         opaque RandomAccessCollection where Self: RandomAccessCollection
         opaque MutableCollection where Self: MutableCollection {
    return ReversedCollection<Self>(...)
  }
}
```

Here, the opaque result type conforms to `MutableCollection` when the `Self` type conforms to `MutableCollection`. This conditional result is independent of whether the opaque result type conforms to `RandomAccessCollection`.

## Detailed design

### Grammar of opaque result types

The grammatical productions for opaque result types are straightforward:

```
type ::= opaque-type opaque-type*
     |   '_'
     
opaque-type ::= 'opaque' type where-clause[opt]
```

The first `type` production allows a sequence of `opaque-type` clauses. Each of those is `opaque` followed by a type (that is semantically restricted to an existential) and an optional `where` clause. The second `type` production introduces the contextual type `_` to describe the opaque result type.

### Restrictions on opaque result types

Opaque result types can only be used within the result type of a non-local function, the type of a variable, or the result type of a subscript. For example, one can return an optional opaque result type:

```swift
func f(flip: Bool) -> (opaque P)? {
  if flip {
    return 1       // concrete type for the opaque result type is Int
  }
  
  return nil
}
```

Opaque result types cannot be used in the requirements of a protocol:

```swift
protocol Q {
  func f() -> opaque P        // error: cannot use opaque result type within a protocol
}
```

Associated types provide a better way to model the same problem, and the requirements can then be satisfied by a function that produces an opaque result type.

Similarly to the restriction on protocols, opaque result types cannot be used for a non-`final` declaration within a class:

```swift
class C {
  func f() -> opaque P { ... }         // error: cannot use opaque result type with a non-final method
  final func g() -> opaque P { ... }  // okay
}
```

Contextual named types (e.g., `_.Element`) can only be used within the `where` clause of an opaque result type; elsewhere, the `_` has no meaning for a type. [Generalized existentials](https://github.com/austinzheng/swift-evolution/blob/az-existentials/proposals/XXXX-enhanced-existentials.md) are likely to expand on this syntax.

### Single opaque result type per entity
As a (possibly temporary) restriction, a particular function/variable/subscript can only contain a single opaque result type, so the following is ill-formed:

```swift
func g(flip: Bool) -> (opaque P, opaque P) {  // error: two opaque result types
  return (1, "hello")
}
```

While it is technically possible to support multiple opaque result types in a given function/variable/subscript, the contextual named type syntax starts to break down. For example, say we want to return two opaque types that conform to `Collection` but whose element types are the same: 

```swift
func g() -> (opaque Collection where _.Element: Equatable, opaque Collection) 
    where /* Element types of both opaque types are equivalent? */ {
  // ...
}
```

The `_.Element` syntax that works nicely for saying that the `Element` type of the first `Collection` is `Equatable` doesn't allow us to relate that `Element` type to the second `Collection` element type. We would need to invent more syntax, e.g., `where _.0.Element == _.1.Element`.

### Uniqueness of opaque result types

Opaque result types are uniqued based on the function/property/subscript and any generic type arguments. For example:

```swift
func makeOpaque<T>(_: T.Type) -> opaque Any { /* ... */ }
var x = makeOpaque(Int.self)
x = makeOpaque(Double.self)  // error: "opaque" type from makeOpaque<Double> is distinct from makeOpaque<Int>
```

This includes any generic type arguments from outer contexts, e.g.,

```swift
extension Array where Element: Comparable {
  func opaqueSorted() -> opaque Sequence where _.Element == Element { /* ... * }
}

var x = [1, 2, 3]. opaqueSorted()
x = ["a", "b", "c"].opaqueSorted()  // error: opaque result types for [Int].opaqueSorted() and [String].opaqueSorted() differ
```

### Ambiguity with `where` clauses

Opaque result types contain optional trailing `where` clauses, as do function and subscript declarations, leading to a parsing ambiguity. For example:

```swift
func sorted<C: RandomAccessCollection>(_: C) -> opaque RandomAccessCollection
    where C.Element: Comparable, _.Element == C.Element {
  /* ... */
}
```

Is the `where` clause part of the opaque result type or part of the function itself? The [maximal munch](https://en.wikipedia.org/wiki/Maximal_munch) principle implies that it should be part of the opaque result type, e.g., that this declaration is equivalent to:

```swift
func sorted<C: RandomAccessCollection>(_: C)
     -> (opaque RandomAccessCollection where C.Element: Comparable, _.Element == C.Element) {
  /* ... */
}
```

rather than being part of the declaration itself:

```swift
func sorted<C: RandomAccessCollection>(_: C) -> (opaque RandomAccessCollection)
    where C.Element: Comparable, _.Element == C.Element {
  /* ... */
}
```

Note that the last of these won't compile, because `_.Element` does not make sense outside of the opaque result type.

We propose to follow the maximal munch principle here, associating the `where` clause with the opaque result type, because it is strictly better than the alternative:

* All constraints of the result type are implied constraints of the function itself, so stating the `C.Element: Comparable` constraint as part of opaque result type also makes that a constraint on the generic function as a whole.
* The generic type parameters are already part of the uniqueness criteria for opaque result types, so there is little downside to exposing all of the known capabilities of the generic type parameters in the opaque result type, because the client already has to satisfy those constraints.

To the second point, consider:

```swift
func weirdSorted<C: RandomAccessCollection>(_: C)
  -> (opaque RandomAccessCollection where _.Element: Equatable,
                                        _.Element == C.Element)
  where C.Element: Comparable {
/* ... */
}
```

This could be interpreted to mean that the resulting opaque result type only guarantees that its element type is `Equatable`, even though the `weirdSorted` function only works with collections whose element types are `Comparable`, and the argument and result element types are equivalent. For simplicity, we state that there is no difference between the opaque result types of `weirdSorted()` and any of the former `sorted()` examples, i.e., the entire generic signature of the function is used to describe the capabilities of the opaque result type.

### Implementation strategy

From an implementation standpoint, a client of a function with an opaque result type needs to treat values of that result type like any other resilient value type: its size, alignment, layout, and operations are unknown.

However, when the body of the function is known to the client (e.g., due to inlining or because the client is in the same compilation unit as the function), the compiler's optimizer will have access to the specific concrete type, eliminating the indirection cost of the opaque result type.

## Source compatibility

Opaque result types are purely additive. They can be used as a tool to improve long-term source (and binary) stability, by not exposing the details of a result type to clients.

If opaque result types were to be adopted in the standard library, it would initially break source compatibility (e.g., if types like `EnumeratedSequence`, `FlattenSequence`, and `JoinedSequence` were removed from the public API) but could provide longer-term benefits for both source and ABI stability because fewer details would be exposed to clients. There are some mitigations for source compatibility, e.g., a longer deprecation cycle for the types or overloading the old signature (that returns the named types) with the new signature (that returns an opaque result type).

## Effect on ABI stability

Opaque result types are an ABI-additive feature, requiring no additional support from the Swift runtime. Changing any of the APIs in the standard library to make use of opaque result types would be an ABI-breaking change, however, so the standard library would have to carefully stage such changes.

## Effect on API resilience

Opaque result types are part of the result type of a function/type of a variable/element type of a subscript. The requirements that describe the opaque result type cannot change without breaking the API/ABI. However, the underlying concrete type *can* change from one version to the next without breaking ABI, because that type is not known to clients of the API.

One notable exception to the above rule is `@inlinable`: an `@inlinable` declaration with an opaque result type requires that the underlying concrete type be `public` or `@usableFromInline`. Moreover, the underlying concrete type *cannot be changed* without breaking backward compatibility, because it's identity has been exposed by inlining the body of the function. That makes opaque result types somewhat less compelling for the `compactMap` example presented in the introduction, because one cannot have `compactMap` be marked `@inlinable` with an opaque result type, and then later change the underlying concrete type to something more efficient.

We could allow an API originally specified using an opaque result type to later evolve to specify the specific result type. The result type itself would have to become visible to clients, and this might affect source compatibility, but (mangled name aside) such a change would be resilient.

## Rust's `impl Trait`

The proposed Swift feature is largely based on Rust's `impl Trait` language feature, described by [Rust RFC 1522](https://github.com/rust-lang/rfcs/blob/master/text/1522-conservative-impl-trait.md) and extended by [Rust RFC 1951](https://github.com/rust-lang/rfcs/blob/master/text/1951-expand-impl-trait.md). There are only a small number of differences between this feature as expressed in Swift vs. Rust's `impl Trait` as described in RFC 1522:

* Swift's need for a stable ABI necessitates translation of opaque result types as resilient types, which is unnecessary in Rust's model, where the concrete type can always be used for code generation.
* Swift's opaque result types are fully opaque, because Swift doesn't have pass-through protocols like Rust's `Send` trait, which simplifies the type checking problem slightly.
* Due to associated type inference, Swift already has a way to "name" an opaque result type in some cases.
* We're not even going to mention the use of `opaque` in argument position, because it's a distraction for the purposes of this proposal; see [Rust RFC 1951](https://github.com/rust-lang/rfcs/blob/master/text/1951-expand-impl-trait.md).
* Rust [didn't tackle the issue of conditional constraints](https://github.com/rust-lang/rfcs/blob/master/text/1522-conservative-impl-trait.md#compatibility-with-conditional-trait-bounds).

## Future Directions

### Opaque type aliases

Opaque result types are tied to a specific declaration. They offer no way to state that two related APIs with opaque result types produce the *same* underlying concrete type. For example, the `LazyCompactMapCollection` type proposed in [SE-0222](https://github.com/apple/swift-evolution/blob/master/proposals/0222-lazy-compactmap-sequence.md) is used to describe four different-but-related APIs: lazy `compactMap`, `filter`, and `map` on various types.

Opaque type aliases allow us to provide a named type with stated capabilities, for which the underlying implementation type is hidden from clients. For example:

```swift
public typealias LazyCompactMapCollection<Elements, ElementOfResult>:
  opaque Collection where _.Element == ElementOfResult
    = LazyMapSequence<
           LazyFilterSequence<
             LazyMapSequence<Elements, ElementOfResult?>
           >,
           ElementOfResult
         >
```

The opaque result type following the `:` is how clients see `LazyCompactMapCollection`. The underlying concrete type, spelled after the `=`, is visible only to the implementation (see below for more details).

Now, multiple APIs can be describing as returning a `LazyCompactMapCollection`:

```swift
extension LazyMapCollection {
	public func compactMap<U>(_ transform: @escaping (Element) -> U?) -> LazyCompactMapCollection<Base, U> {
	  // ...
	}

	public func filter(_ isIncluded: @escaping (Element) -> Bool) -> LazyCompactMapCollection<Base, Element> {
	  // ...
	}
}
```

From the client perspective, both APIs return the same type, but the specific underlying type is not known.

```swift
var compactMapOp = values.lazy.map(f).compactMap(g)
if Bool.random() {
  compactMapOp = values.lazy.map(f).filter(h)  // okay: both APIs have the same type
}
```

The underlying concrete type of an opaque type alias has restricted visibility. It's access is the more-restrictive of the access level below the type alias's access (e.g., `internal` for a `public` opaque type alias, `private` for an `internal` opaque type alias) and the access levels of any type mentioned in the underlying concrete. For the opaque type alias `LazyCompactMapCollection` above, this is the most restrictive of `internal` (one level below `public`) and the types involved in the underlying type (`LazyMapSequence`, `LazyFilterSequence`), all of which are public. Therefore, the access of the underlying concrete type is `internal`.

If instead the concrete underlying type of `LazyCompactMapCollection` involved a private type, e.g.,

```swift
private struct LazyCompactMapCollectionImpl<Elements: Collection, ElementOfResult> {
  // ...
}

public typealias LazyCompactMapCollection<Elements, ElementOfResult>:
  opaque Collection where _.Element == ElementOfResult
    = LazyCompactMapCollectionImpl<Elements, ElementOfResult>
```

then the access of the underlying concrete type would be `private`.

The access of the underlying concrete type only affects the type checking of function bodies. If the function body has access to the underlying concrete type, then the opaque typealias and its underlying concrete type are considered to be equivalent. Extending the example above:

```swift
extension LazyMapCollection {
  public func compactMap<U>(_ transform: @escaping (Element) -> U?) -> LazyCompactMapCollection<Base, U> {
    // okay so long as we are in the same file as the opaque type alias LazyCompactMapCollection,
    // because LazyCompactMapCollectionImpl<Base, U> and
    // LazyCompactMapCollection<Base, U> are known to be identical
    return LazyCompactMapCollectionImpl<Base, U>(elements, transform)
  }
}
```

Opaque type aliases often an alternative solution to the issue with conditional conformance. Instead of annotating the `opaque` result type with all of the possible conditional conformances, we can change `reversed()` to return an opaque type alias `Reversed`, giving us a name for the resulting type:

```swift
public typealias Reversed<Base: BidirectionalCollection>:
  opaque BidirectionalCollection where _.Element == Base.Element
    = ReversedCollection<Base>
```

Then, we can describe conditional conformances on `Reversed`:

```swift
extension Reversed: RandomAccessCollection where Element == Base.Element { }
```

The conditional conformance must be satisfied by the underlying concrete type (here, `ReversedCollection`), and the extension must be empty: `Reversed` is the same as `ReversedCollection` at runtime, so one cannot add any API to `Reversed` beyond what `ReversedCollection` supports.
