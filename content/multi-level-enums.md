---
layout: sip
permalink: /sips/:title.html
stage: implementation
status: waiting-for-implementation
presip-thread: https://contributors.scala-lang.org/t/pre-sip-foo-bar/9999
title: SIP-NN - Multi-Level Enums
---

# SIP: Multi-Level Enums

**By: Raphael Bosshard**

## History

| Date       | Version            |
| ---------- | ------------------ |
| 2025-09-15 | Initial Draft      |
| 2026-02-17 | Revised Draft      |



## Summary

This proposal adds minimal syntax to allow enums with nested enumerations, maintaining full exhaustivity checking while keeping syntax clean and intuitive.

## Motivation

Scala 3 introduced `enum` as a concise and type-safe way to define algebraic data types. However, it currently supports only flat enums. Many real-world use cases, such as domain modeling or UI state machines, naturally require **hierarchical enums**, where cases are grouped into logical families or categories.

Consider modeling animals:

```scala
enum Animal:
  case Dog, Cat, Sparrow, Penguin
```

This flat structure works, but grouping `Dog` and `Cat` under `Mammal`, and `Sparrow` and `Penguin` under `Bird` is more expressive and enables cleaner abstraction.


## Proposed solution

### High-level overview

Allow `enum` cases to contain **nested `enum` definitions**, using a consistent indentation-based syntax.

```scala
enum Animal:
  case enum Mammal:
    case Dog, Cat
  case enum Bird:
    case Sparrow, Pinguin
```

Each nested `enum` case defines a group of related subcases. The **nested enum is itself a valid subtype** of the parent enum, and its members are **valid cases** of the parent enum, allowing full exhaustivity and pattern matching.

### Specification

#### Enum Definition:

```scala
enum Animal:
  case enum Mammal:
    case Dog, Cat
  case enum Bird:
    case Sparrow, Pinguin
  case Fish
```

- `case enum Mammal:` introduces a **sub-enum case**.
- Nested cases (`Dog`, `Cat`) are **automatically part of the parent enum** (`Animal`), as well as part of the sub-enum (`Mammal`).

#### Desugaring / Type Relationships


1. An `enum E` lowers to a sealed abstract class `E` plus a companion `object E`.
2. A nested `case enum S` declared inside `E` lowers to a sealed abstract class `S extends E` and a companion `object S` placed in `E`'s companion.
3. Each nested leaf `case` lowers to a `case object` (or `case class` for parameterized cases) that extends the innermost declared abstract class (the immediately enclosing `case enum`).
4. If a `case` has no nested subcases it is a leaf and becomes a direct subclass of the enclosing enum class.

Example lowered shape (conceptual):

```scala
sealed abstract class Animal
object Animal:
  sealed abstract class Mammal extends Animal
  object Mammal:
    case object Dog extends Mammal
    case object Cat extends Mammal

  sealed abstract class Bird extends Animal
  object Bird:
    case object Sparrow extends Bird
    case object Pinguin extends Bird

  case object Fish extends Animal
```

Notes / rules:

- Enums are modeled as sealed abstract classes; all subtype relations asserted by nested `case enum` follow that model.
- `extends` clauses on cases follow normal Scala rules: a case may extend an accessible abstract class or trait; when it names the enclosing supercase class (the immediately enclosing `case enum`), that creates the intended parent/child enum subtype relation.
- Constructor expressions of nested leaf cases are "widened" to the immediately enclosing enum type (the nearest enclosing `enum` or `case enum` under which the case is declared). In other words, the declared parent is the target widening type.
- A nested `case enum` that contains subcases is not simultaneously a distinct term-case value of its parent. If it has subcases it represents a namespace/type; to have both a singleton term and subcases you must declare them separately with distinct names.

- Type relationships: for a nested declaration `case enum S` inside `enum E`, every leaf `L` declared under `S` satisfies `L <: S <: E`. Parameterized cases keep their declared type parameters and variance; companion objects are singletons whose `.type` is a subtype of the declared class type.

```scala
enum Animal:
  case enum Mammal:
    case Dog, Cat
  case Fish
```

// Lowered/type relations
// Dog <: Mammal <: Animal
// Cat <: Mammal <: Animal
// Fish <: Animal

// Widening (constructor expression widens to the nearest enclosing enum):
```scala
val d: Mammal = Animal.Mammal.Dog
val a: Animal = Animal.Mammal.Dog
```

// `.values` and `ordinal` (per-enum view):
Animal.values == Array(Dog, Cat, Fish)
Dog.ordinal in Animal == 0
Cat.ordinal in Animal == 1
Fish.ordinal in Animal == 2

Mammal.values == Array(Dog, Cat)
Dog.ordinal in Mammal == 0

// Matching semantics:
a match
  case _: Mammal => // matches Dog | Cat
  case Mammal  => // matches singleton Mammal only when declared as a leaf

// Inferred term-subcase type:
val x = Animal.Mammal.Dog // inferred: Animal.Mammal.Dog.type

// Prohibited ambiguity (example):
enum Bad:
  case Mammal           // term
  case enum Mammal:     // error: duplicate/ambiguous name
    case Dog
```
```scala
enum Animal:
  case enum Mammal:
    case Dog, Cat
  case Fish

// Lowered/type relations
// Dog <: Mammal <: Animal
// Cat <: Mammal <: Animal
// Fish <: Animal

// Widening (constructor expression widens to the nearest enclosing enum):
val d: Mammal = Animal.Mammal.Dog
val a: Animal = Animal.Mammal.Dog

// `.values` and `ordinal` (per-enum view) — valid Scala checks:
assert(Animal.values.toList == List(Dog, Cat, Fish))
assert(Animal.values.indexOf(Dog) == 0)
assert(Animal.values.indexOf(Cat) == 1)
assert(Animal.values.indexOf(Fish) == 2)

assert(Mammal.values.toList == List(Dog, Cat))
assert(Mammal.values.indexOf(Dog) == 0)

// Matching semantics:
a match
  case _: Mammal => // matches Dog | Cat
  case Mammal    => // matches singleton Mammal only when declared as a leaf

// Inferred term-subcase type:
val x = Animal.Mammal.Dog // inferred: Animal.Mammal.Dog.type

// Prohibited ambiguity (example):
enum Bad:
  case Mammal           // term
  case enum Mammal:     // error: duplicate/ambiguous name
    case Dog
```


#### Pattern Matching

Exhaustive pattern matching on `Animal` must cover all leaf cases:

```scala
def classify(a: Animal): String = a match
  case Dog     => "a dog"
  case Cat     => "a cat"
  case Sparrow => "a bird"
  case Pinguin => "a penguin"
  case Fish    => "a fish"
```

Matching on intermediate enums is allowed, but the pattern form matters:

```scala
def isWarmBlooded(a: Animal): Boolean = a match
  case _: Mammal => true     // matches any value whose runtime type conforms to Mammal (covers Dog, Cat)
  case Bird     => true     // matches the singleton term `Bird` (only if `Bird` is declared as a term value)
  case Fish     => false
```

Semantics:

- `case _: Mammal` matches any leaf whose type is a subtype of `Mammal`.
- `case Mammal` (no `_:`) matches the singleton term `Mammal` only when `Mammal` is itself a declared term-case (i.e. a leaf); it is not shorthand for all subcases.


#### `values`, `ordinal`, `valueOf`

- `.values` for an enum returns the leaf cases that belong to that enum (the enumeration's view of leaves). Example: `Animal.values == [Dog, Cat, Sparrow, Pinguin, Fish]` and `Mammal.values == [Dog, Cat]`.
- `ordinal` is defined relative to the enum whose `.values` is considered: a leaf's `ordinal` in `E` is its index in `E.values`.
- Non-leaf groupings (i.e. a `case enum` that contains subcases) are not themselves entries of their parent `.values`; therefore they do not receive ordinals in the parent's ordering. A name that is both a singleton value and a container is disallowed to avoid ambiguity.
- `valueOf` works against the `.values` set of the enum being queried.

Proposal for term-subcase typing:

- For expressions like `val x = Animal.Mammal.Dog` the inferred type should be the singleton type `Animal.Mammal.Dog.type` (the most precise static type), which also conforms to `Mammal` and `Animal`.


#### Sealed-ness and Exhaustivity

- The parent enum and all nested enums are sealed.
- Pattern matching at any level (e.g. on `Mammal`) is **exhaustive** at that level.
- At the top level (`Animal`), exhaustivity means all leaf cases must be covered.

#### Reflection / Enum APIs

- `Animal.values`: All leaf values (`Dog`, `Cat`, `Sparrow`, etc.)
- Each nested `case enum` (e.g., `Mammal`) gets its own `.values`, `.ordinal`, and `.valueOf` API.
- `ordinal` value of leaves are global to the supercase  (e.g., `Dog.ordinal == 0`, `Cat.ordinal == 1`, etc.)

#### Syntax Specification (EBNF-like)

```
EnumDef         ::= 'enum' Id ':' EnumBody
EnumBody        ::= { EnumCase }
EnumCase        ::= 'case' EnumCaseDef
EnumCaseDef     ::= Ids
                 | 'enum' Id ':' EnumBody
Ids             ::= Id {',' Id}
```


#### Compiler

- The Scala compiler must treat nested enums inside `case enum` as part of the parent enum’s namespace.
- Exhaustivity checking logic must recursively analyze nested enums to extract leaf cases.
- Type relationships must be modeled to reflect subtyping: e.g. `Dog <: Mammal <: Animal`.


#### Examples

#### Example 1: Basic Structure

```scala
enum Shape:
  case enum Polygon:
    case Triangle, Square

  case enum Curve:
    case Circle

  case Point
```

#### Example 2: Moddeling size information

```
enum SizeInfo {
  case Bounded(bound: Int)
  case enum Atomic {
    case Infinite
    case Precise(n: Int)
  }
}
```

#### Example 3: Generalized `Either`

```
enum AndOr[+A, +B] {
  case Both[+A, +B](left: A, right: B) extends AndOr[A, B]
  case enum Either[+A, +B] extends AndOr[A, B] {
    case Left[+A, +B](value: A) extends Either[A, B]
    case Right[+A, +B](value: B) extends Either[A, B]
  }
}

```

#### Example 4:
Grouping JSON values into primitives and non-primitives

```
enum JsValue {
  case Obj(fields: Map[String, JsValue])
  case Arr(elems: ArraySeq[JsValue])
  case enum Primitive {
    case Str(str: String)
    case Num(bigDecimal: BigDecimal)
    case JsNull
    case enum Bool(boolean: Boolean) {
      case True extends Bool(true)
      case False extends Bool(false)
    }
  }
}
```




### Compatibility
- Fully backwards compatible: does not affect existing flat enums.
- Adds optional expressiveness.
- Libraries using `enum` APIs (e.g., `values`) will continue to function with leaf-only views.
- Mirrors and macros

### Other Concerns

#### Macro / Tooling Support

- IDEs and macros need to understand the nested structure.
- Pattern matching hints and auto-completion should support matching on intermediate cases.


### Feature Interactions


### Alternatives

- **Flat enums with traits**: more verbose, less exhaustivity checking, more boilerplate.
- **Nested cases with `extends`**: heavier syntax, harder to teach/read.
- **DSLs or macros**: non-standard, cannot integrate with Scala's `enum` semantics cleanly.


## Related Work


## Faq


