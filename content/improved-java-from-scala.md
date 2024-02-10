---
layout: sip
permalink: /sips/:title.html
stage: implementation
status: waiting-for-implementation
title: SIP-NN - Title of the Proposal
---

**By: Yilin Wei**

## History

| Date          | Version            |
|---------------|--------------------|
| Feb 10th 2024 | Initial Draft      |

## Summary

This proposal aims to improve Scala's interoptability with Java's records.

It proposes to allow Java records to be pattern matched and automatic synthesis of mirrors.

## Motivation

Since [JEP-395](https://openjdk.org/jeps/395) landed, Java has had a less-powerful version of Scala's case classes, called records. Together with improved pattern support in switch statements which landed in [JEP-440](https://openjdk.org/jeps/440), users of Java are more familiar than ever with pattern matching.

Suppose we have a user, Alice, who maintains a Java application which makes use of records. The coding style would be remarkably similar to the equivalent Scala code — favouring immutability, structural recursion to implement the logic of the program. Alice has heard of Scala and learns that Scala is meant to be _even_ more ergonomic for the set of features which she uses in Java. She wishes to introduce the language to her team; primarily switching the parts of the code base which are already amenable to the style of programming which Scala excels at.

However, moving a codebase is a daunting task; doing it incrementally means that such a decision is more easily made since it is less risky. If Scala does not live up to her expectations (we hope it will!), then she can always revert the changes.

Broadly speaking, Alice realises that she has two options. 

The first is to rewrite all her domain objects to case classes; this primarily concerns interoptability in the _other_ direction i.e. interoptability in calling Scala code from Java. A discussion about future work and what this could look like is discussed later but are outside of the scope of this SIP.

The second, is to keep the domain objects she has currently we are referenced all over the code base but start translating her logic into Scala. Since The same code would look _very_ similar in Scala, she chooses this option.

She then starts to mechanically translate an algorithm from:

~~~java
public int magnitude(Point p) {
  switch(p) {
    case Point(int x, int y) -> x + y;
  }
}
~~~

to her imagined equivalent in Scala:

~~~scala
def magnitude(p: Point) = p match:
  case Point(x, y) => x + y
~~~

but quickly realises that her nice algorithms fail to compile since pattern matching on records is not supported. However, she looks through the specification and sees that she may write custom [extractors](https://docs.scala-lang.org/scala3/reference/changed-features/pattern-matching.html) to support her domain objects.

Given a domain object her domain object written in Java:

~~~java
record Point(int x, int y) {}
~~~

A name-based matcher solution (which would not allocate), would look like the following:

~~~scala
final class PointNames(p: Point) extends AnyVal:
  def _1: Int = p.x
  def _2: Int = p.y

final class PointUnapply(p: Point) extends AnyVal:
  def isEmpty: false = false
  def get: PointNames = new PointNames(p)
  
object PointExtractor:
  def unapply(p: Point): PointUnapply = PointUnapply(p)
~~~

Or a tuple-based solution, which would allocate and box more than the equivalent Java code, but requires less boilerplate.

~~~scala
object PointExtractor:
   def unapply(p: Point): (Int, Int) = (p.x, p.y)
~~~

Her algorithm now looks more similar to what she wanted.

~~~scala
def magnitude(p: Point) = p match:
  case PointExtractor(x, y) => x + y
~~~

However, the process has not been ergonomic. If a team member, other than Alice, opens their IDE and starts to navigate around the codebase, they would quickly be exposed to the implementation details — Alice would then have to explain the more advanced features and nuances of extractor objects to her team members.

## Proposed solution

We propose that the compiler be updated to allow pattern matching on records and do automatic synthesis of a `Mirror.Product` for all records.

Before presenting the solution, we will first include a brief primer of records. The relevant [JEP-395](https://openjdk.org/jeps/395) introduces several terms which we are concerned with. The first term is the _canonical constructor_ which is the constructor defined in the _header_ of the record class; this is similar to the `primaryConstructor` which we define for case classes. The second term are the _components_ of the record — these are the fields defined in the header. A record extends `java.lang.Record` and is implicitly `final` and the fields are immutable. A user can override field definitions; semantically, we must always go via the exposed public accessors.

For the `Point` record described earlier:

```java
record Point
  // This is the header
  (
   // These are component
   int x, int y) {

   // This is an example of a component override
   @Override
   public int x() {
     return x + 1
   }
}
```

### Pattern matching

We propose generating a new synthetic `unapply` method on the companion module of the record returning itself - the same synthesized implementation for case classes introduced in Scala 3. This `unapply` method should have an `inline` modifier, so that the compiled byte code would not refer to it. There is already a precedent; the compiler currently generates a synthesized `inline` `apply` method for all constructors; including constructors for Java classes.

The compiler would be updated to allow argument patterns for `unapply` methods which _return_ a record. Each argument pattern would bind to the corresponding accessor in the order of the _canonical constructor_. We reject an `UnApply` node which does not have the same arity as the canonical constructor. The semantics for Scala's argument patterns are similar enough to Java's own pattern matches in `switch` statements that the equivalent code in Java and Scala can be correlated by a relative beginner to the language.

~~~java
switch (p) {
  case Point(int x, int y) -> x + y
}
~~~

And the equivalent Scala code,

~~~scala
p match:
  case Point(x, t) => x + y
~~~

We also allow the pattern to appear in all other locations which currently support it, such as when defining a value:

~~~scala
val Point(x, _) = p
~~~

This will also allow users to return a record class for their own custom extractors in places where a `Product` is allowed in the current specification.

### Synthetic mirror

Currently, the Scala compiler generates a synthetic [mirror](https://docs.scala-lang.org/scala3/reference/contextual/derivation.html#mirror-1) for case classes. The `Mirror` contains metadata about the product, and is used by the ecosystem for type class derivation at compile time. Conceptually speaking, a record is also a product type and should have a mirror associated with it.

We propose generating a synthetic `Mirror.ProductOf` for a record when requested. We expect the type of the mirror to be the same as if the record was the equivalent case class; with the `MirrorElemTypes` and `MirrorElemLabels` corresponding to the components of the record. For the `Point` record, a mirror conforming to the following interface would be generated.

~~~scala
trait Mirror.Product:
  type MirrorType = Point
  type MirrorElemTypes = (int, int)
  type MirrorLabel = "Point"
  type MirroredMonoType = Point
  type MirrorElemLabels = ("x", "y")
  
  def fromProduct(p: Product): MirroredMonoType
~~~

Unfortunately, we cannot use the strategy used by case classes in the compiler which currently add an extra method to the companion object and using that as the mirror. Therefore, we propose an addition to the _runtime_ library with an implementation of a mirror with a `fromProduct` which would make use of Java's reflection capabilities to construct an arbtrary record dynamically from a product.

### High-level overview

The following examples would now compile:

~~~ scala
val Point(x, _) = p
~~~

~~~ scala
p match:
  case Point(x, y) => x + y
~~~

~~~ scala
summon[Mirror.ProductOf[Point]]
~~~

### Specification

TBD

### Compatibility

#### Impact of synthetic `Mirror`

It should be noted, that the current eco-system implicitly assumes that a `Mirror.Product` returns a class which extends `Product` even though there are no type bounds on the actual trait itself. 

This is implied by the [example](https://docs.scala-lang.org/scala3/reference/contextual/derivation.html#how-to-write-a-type-class-derived-method-using-low-level-mechanisms-1) in the official documentation and can be seen in the source code of well-used libraries in the wild. 

Circe, a JSON encoding library, does an explicit cast from an arbitrary type `A` to `Product` to encode an object which has a mirror. The relevant code can be seen [here](https://github.com/circe/circe/blob/26ab88aab323fe2a2c2afd3dadd64dc875d865db/modules/core/shared/src/main/scala-3/io/circe/derivation/ConfiguredEncoder.scala#L31). Releasing the changes proposed would mean that a codec would be able to be derived for a record, but the program would fail at runtime — which is not ideal behaviour.

On the other hand, current programs which are valid would behave exactly the same, with no changes semantically. Library authors would be encouraged to do a minor update to enable users to derive typeclasses for records.

Unfortunately, the [`Record`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Record.html) class lacks a general method to get the field by arity. For that reason, the author recommends adding the following extension methods to the standard library to ease the migration.

~~~scala
extension [T <: Record : ClassTag](t: T):
  def productArity: Int = ...
  def productElement(i: Int): Any = ...
~~~

This would make use of the `getRecordComponent` and associated reflection facilities in to access arbitrary members of the record.

Another potential alternative is to add the methods `productArity` and `productElement` as methods onto the mirror. This would have the additional benefit of being able to cache the reflective calls on the actual mirror object so that the calls to access the components was less expensive.

All other usages of the mirror — such as using the macro [reflection API](https://dotty.epfl.ch/docs/reference/metaprogramming/reflection.html) to access field members via labels, inspect the types or reify the [labels themselves](https://github.com/MateuszKubuszok/dbg/blob/3d32b63211dd88a0b396b249aca2b1a8c6ea721f/src/main/scala/dbg/schema/Dbg.scala#L130) should work out of the box with the derived mirror of records or fail to compile.

Existing program semantics are preserved.

### Other concerns

### Open questions

#### Addition of JDK 16+ dependent class into standard library

TBD

#### Synthesizing a complete mirror at each callsite

TBD

## Alternatives

### Tuple-based pattern matching

The simplest encoding creates an intermediate `Tuple` of the correct arity for the record class. For the `Point` record discussed earlier, we would synthetisize the following function:

~~~scala
inline def unapply(p: Point): (Int, Int) = (p.x, p.y)
~~~

This has the advantage of requiring no changes in the pattern matcher, but the disadvantage of requiring more allocations and not allowing the user to return a record for a custom extractor. The relatively low amount of extra code to support the records directly in the pattern matcher meant that the author rejected this solution.

It is worth noting that the author tried using the option-less encoding as well, but ran into issues with [inlining](https://github.com/lampepfl/dotty/pull/19631).

## Related work

A WIP implementation can be found [here](https://github.com/lampepfl/dotty/pull/19577) for interested parties. It compiles all the examples listed above. The same implementation was outlined in [July 2023](https://github.com/lampepfl/dotty/discussions/18115) by Guillaume Matres.

### Further work

#### Calling Scala from Java

So far this proposal has only concerned itself in one direction. The opposite direction — pattern matching Scala case classes from Java has also had interest from the [community](https://contributors.scala-lang.org/t/javas-record-is-faster-can-scala-leverage-it/6318/21).

#### Sealed classes

This proposal only discusses Java's version of products but not the equivalent of sums [JEP-409](https://openjdk.org/jeps/409). Ideally Scala's exhaustivity matching and mirrors would also extend to support sealed classes in some form.

## FAQ
