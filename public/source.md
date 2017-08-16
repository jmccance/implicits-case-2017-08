name: title
layout: true
class: center, middle, section
---
template: title

# Implicits in Scala

Joel McCance  
[@jmccance](http://twitter.com/jmccance)  
17 August 2017

---
layout:false

---
class: middle

> "If there's one feature that makes Scala 'Scala', I would pick implicits. There's hardly an API without them. They enable advanced and elegant architectural designs. They are also misused way too often."  
  <cite><strong>Martin Odersky</strong>, Scala Days Chicago Keynote, 2017</cite>

---
# Overview

1. Implicits 101
2. Uses and Abuses
3. Implicit Functions

---
template: title

# Implicit Taxonomy

---

# Types of Implicits

- Implicit parameters
- Implicit values
- Implicit conversions
- Implicit classes

---
layout: true

# Implicit Parameters

---

```scala
def doStuff(foo: Foo, context: Context): Bar
def getStuff(context: Context): Stuff
```

---

```scala
def doStuff(foo: Foo)(implicit context: Context): Bar
def getStuff()(implicit context: Context): Stuff
```

--

For a function to accept implicit parameters, it must...

- Be a method (a `def`)
- And have a parameter list marked with the `implicit` keyword, which may have one or more arguments
- The implicit parameter list must be the last parameter list on the method

---

Good

```scala
def foo(implicit bar: Bar): Baz = ???
    // Invoked as `foo`
def foo()(implicit bar: Bar, baz: Baz): Quux = ???
    // Invoked as `foo()`
def foo(bar: Bar)(implicit baz: Baz): Quux = ???
    // Invoked as `foo(bar)`
def foo(bar: Bar)(baz: Baz)(implicit quux: Quux): Wibble = ???
    // Invoked as foo(bar)(baz)
```

--

Not Good

```scala
def foo(implicit bar: Bar)(baz: Baz): Quux = ??? 
    // Implicit parameters must be last
val foo: Bar => Baz = { implicit bar => ??? }
    // Implicit parameters will not be passed to functions (...yet)
    // But this _will_ compile!
```

---

---
layout: true

# Implicit Values

---
layout: true

# Implicit Conversions

---
layout: true

# Implicit Classes

---
layout: false
class: middle

> Okay, but where the @*#$ do these things come from, exactly?

---
layout: true

# The Implicit Scope
---

1. "All identifiers _x_ that can be accessed at the point of the method call without a prefix and that denote an implicit definition or an implicit parameter."
2. "All `implicit` members of some object that belongs to the implicit scope of the implicit parameter's type _T_.

  The _implicit scope_ of a type _T_ consists of all companion modules of classes that are associated with the implicit parameter's type."

.footnote["Implicits", [Scala Language Specification](http://www.scala-lang.org/files/archive/spec/2.12/07-implicits.html)]

---

## Option 1: The Current Scope

- Is it defined in the current scope? (E.g., this function, this class.)
- Is it in the package object of the current package?
- Is it being imported?

---

## Option 2: Companion Objects

### Implicit Conversions

Is it in the companion object of the _source_ or _target_ type?

```scala
object Foo {
  // If you try to use a Foo where you need a Bar, this will be used.
  implicit def toBar(foo: Foo): Bar = ???

  // If you try to use a Bar where you need a Foo, this will be used.
  implicit def fromBar(bar: Bar): Foo = ???
}

val bar: Bar = someFoo // => toBar(someFoo)
val foo: Foo = bar // => fromBar(bar)
```

---

## Option 2: Companion Objects

### Implicit Values

Is it in the companion object of the top-level type?

```scala
val n: read[Int](s)
val isP = read[Boolean](s)

object Read {
  implicit val readInt: Read[Int] = ???
  implicit val readboolean: Read[Boolean] = ???
}
```

---
layout: false

## The Implicit Scope: Recap

- The search runs in two phases: immediate (local definitions, direct imports), then the implicit scope (companion objects of involved types).
- If there are multiple candidates found in a given scope, the implicit is ambiguous and compilation fails.
- Relevant types: the type, any type parameters, and (for conversions) the expected type.

---
template: title

# Uses and Abuses

---
template: title

# Implicit Functions

---
layout: false

# References

- Odersky, "Implicit Function Types", [scala-lang.org](https://www.scala-lang.org/blog/2016/12/07/implicit-function-types.html)
- Odersky, et al., "Scala Language Specification, Version 2.12", [scala-lang.org](http://www.scala-lang.org/files/archive/spec/2.12/)
- Odersky, "What to Leave Implicit", Scala Days Chicago 2017, [YouTube](https://youtu.be/Oij5V7LQJsA)