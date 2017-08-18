name: title
layout: true
class: center, middle, title
---
template: title

# Implicits in Scala

Joel McCance  
[@jmccance](http://twitter.com/jmccance)  
17 August 2017

???

Introduction. Name. Senior Engineer here at Pandera Labs, as of ~80h ago. In my past work, been lucky enough to work on a few Scala projects professionally, usually with teams completely new to the language. Had the opportunity to help introduce Scala to a lot of newbies, with all its quirks and novelties.

I think implicits are one of the things that tend to surprised and confuse the most. Don't really exist in most languages. Lots of languages have objects. Functions. Inheritance. Lambdas. I've only ever heard of implicits in Scala and Agda, though I'm sure there are others.

---
layout: false
class: middle

> "If there's one feature that makes Scala 'Scala', I would pick implicits. There's hardly an API without them. They enable advanced and elegant architectural designs. They are also misused way too often."  
  <cite>**Martin Odersky**, Scala Days Chicago Keynote, 2017</cite>

???

So implicits are something most developers have never seen, but they permeate a huge amount of the Scala ecosystem. For a lot of people, they're still something of a mystery. So I wanted to use this talk to walk through the basics of implicits, how you're likely to see them used (and misused), and spend a little bit of time talking about a new kind of implicit coming in the future.

---
# Overview

1. Implicits 101
2. Uses and Abuses
3. Working with Implicits
3. Implicit Functions

???

- Implicits 101. Some might be familiar, but I find there's usually a few parts of the picture that each developer is missing.

---
template: title

# Motivation

???

Say you wanted to write an API that made it easier to read data types from Strings.

---
layout: true

# Motivation
---

"I want to be able to read `A`s from strings."

```scala
trait Read[A] {
    def read(s: String): A
}
```

???

You might write a type something like this. (We'll ignore error-handling for now.) Just a simple type that gives you a function from `String`s to `A`s.

Let's look at how you might use this.

---

"So far, so good."

```scala
val readFoo: Read[Foo] = ???

val myFoo = fooRead.read(s) // s: String
```

---

"Look, it's parameterized! I'm such a good functional programmer."

```scala
def loadFromFile[A](file: File, readA: Read[A]): Seq[A] = ???

val foos: Seq[Foo] = loadFromFile(someFile, readFoo)
```

---

"They even compose nicely. This is great!"

```scala
def readSeq[A](readA: Read[A]): Read[Seq[A]] = ???

val foos = seqRead(fooRead).read(s)
```

---

"Uh… this is less great."

```scala
def eitherRead[A, B](readA: Read[A], readB: Read[B]): Read[Either[A, B]] = ???

def mapRead[A, B](readA: Read[A], readB: Read[B]): Read[Map[A, B]] = ???

val myComplexFoo: Map[String, Seq[Either[String, Foo]]] =
    mapRead(stringRead, seqRead(eitherRead(stringRead, fooRead))).read(s)
```
--

.reaction[😒]

???

Wouldn't it be great if the compiler could figure this out for us?

---
template: title

# Implicit Taxonomy

---
layout: false
class: middle

> "Implicits are con-textual. They are _with_ the text, but not _in_ the text."
  <cite>**Martin Odersky**, Scala Days Chicago Keynote, 2017</cite>

???

Implicits give us a tool for accessing information without specifically naming it, in a more controlled way than using global state.

---

# Types of Implicits

- Implicit parameters
- Implicit values
- Implicit conversions
- Implicit classes

---
name: section
layout: true
class: middle, center
---
# Implicit Parameters

---
layout: true

# Implicit Parameters

---

This...

```scala
def doStuff(foo: Foo, context: Context): Bar
def getStuff(context: Context): Stuff
```

--

Becomes this...

```scala
def doStuff(foo: Foo)(implicit context: Context): Bar
def getStuff(implicit context: Context): Stuff
```

---

## The Rules

For a function to accept implicit parameters, it must...

- Be a method (a `def`)
- Have a parameter list marked with the `implicit` keyword, which may have one or more arguments
- The implicit parameter list must be the last parameter list on the method
- Scala will search the "implicit scope" to fill in the missing parameters. (More on this later.)

```scala
def f(a: A, b: B)(implicit c: C, d: D): E = ???
```

---

## Good

```scala
def foo(implicit bar: Bar): Baz = ???
    // Invoked as `foo`

def foo()(implicit bar: Bar, baz: Baz): Quux = ???
    // Invoked as `foo()`

def foo(bar: Bar)(implicit baz: Baz): Quux = ???
    // Invoked as `foo(bar)`

def foo(bar: Bar)(baz: Baz)(implicit quux: Quux): Wibble = ???
    // Invoked as foo(bar)(baz)

def foo[A](a: A)(implicit baz: Bar[A]): Quux[A] = ???
    // Note that you can parameterize implicit parameters.
```

???

- The `foo` version is useful if you want to use context to calculate a value. The API looks like a field.
- The `foo()` version is useful if you want to indicate that this is more of a procedure or a side-effecting method. Note the empty parens.

---

## Not Good

```scala
def foo(implicit bar: Bar)(baz: Baz): Quux = ??? 
    // Implicit parameters must be last

val foo: Bar => Baz = { implicit bar => ??? }
    // Implicit parameters will not be passed to functions (...yet)
    // But this _will_ compile!
```

--

```scala
    // Equivalent to:
val foo: Bar => Baz => { bar =>
  implicit val bar2 = bar
  ????
}
```

???

- Note that you can have any number of parameters, any number of (non-implicit) parameter-lists.
- Error you get for the out-of-place implicit parameter list is not super obvious in IntelliJ, but SBT calls it out nicely:

    ```
    <console>:1: error: an implicit parameter section must be last
         def foo(implicit foo: Foo)(x: Int): Int = ???
    ```
- For the implicit function value case, the reason `implicit` is allowed there is because this marks the argument as implicit. Methods that take implicit parameters will be able to pick this up as if you aliased the argument to an implicit value.

---

Note that you can always pass implicit parameters explicitly if you need or want to.

```scala
def implicit f(a: A)(implicit b: B): C = ???

f(a)(b) // Completely valid
```

???

This will change in the future; you'll have to say `f(a).explicitly(b)`, or similar.

---
layout: false
template: section

# Implicit Values

---
layout: true

# Implicit Values
---

- Implicit values are how you populate implicit parameters.
- Declare an implicit value with `implicit val`.

```scala
implicit val context: Context = ???
```

---

Can also generate the implicit value at runtime with `implicit def`:

```scala
implicit def context: Context = ???
```

---

Implicit values generated from methods can also take implicit parameters:

```scala
implicit def context(implicit otherContext: MoreContext): Context = ???
```

---

Implicit values can be parameterized.

```scala
// "If I know how to read A, I can read Seq[A]."
implicit def readsSeq[A](implicit ra: Reads[A]): Reads[Seq[A]] = ???
```

???

This is a big deal. We'll see later, this is what powers typeclasses.

---
layout: false
template: section

# Implicit Conversions

---
layout: true

# Implicit Conversions

---

I know how much you all like type coercion.

--

```scala
implicit def toInt(s: String): Int = Integer.parseInt(s)

3 * "2" == 6 // true
```

--

.footnote[(Please don't do this.)]

???

Started with this example to show the double-edged sword of conversions. They are powerful, and there are some good use cases for them. But you can also create some major brain-teasers for yourself if you're not careful.

---

## The Rules

Implicit conversions are recipes for the compiler to convert from one type into another.

- Methods or functions declared with the `implicit` keyword.
- They must take exactly one, non-implicit parameter.
- They may optionally take additional, implicit parameters.
- The compiler will only make unambiguous conversions.
- Only apply when there is a type error (unexpected type or missing member).

```scala
implicit def f(a: A): B = ???
implicit val f: A => B = ???
```

---

## Examples

```scala
implicit def f(foo: Foo): Bar = ???
    // Applies to any Foo if this conversion is in scope.

implicit def f(foo: Foo)(implicit baz: Baz): Bar = ???
    // Will only apply if the conversion is in scope and there is an implicit
    // Baz in scope.
```

---
template: section

# Implicit Classes

---
layout: true

# Implicit Classes
---

With implicit conversions, you can implement extension methods on classes.

```scala
class MyStringOps(s: String) {
  def shouted: String = s"${s.toUpperCase}!!!!!"
}

implicit def toMyStringOps(s: String): MyStringOps = new MyStringOps(s)
```

???

Drawbacks:

- Always instantiates a new instance of MyStringOps.
- Verbose

---

Implicit classes tighten up this pattern.

```scala
implicit class MyStringOps(val s: String) extends AnyVal {
    def shouted: String = s"${s.toUpperCase}!!!!!"
}
```

- Same rules apply as for implicit conversions.

???

- AnyVal is optional, but recommmended. Avoids unnecessary instantiations.

---
layout: false

# Implicits 101: Recap

- *Implicit parameters*
    - Arguments to methods that the compiler will provide for you
- *Implicit values*
    - Candidates for use as implicit parameters
- *Implicit conversions*
    - Programmable type coercion
- *Implicit classes*
    - Syntactic sugar for implicit conversions to facilitate writing extension methods

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

## Phase 1: The Current Scope

- Is it defined in the current scope? (E.g., this function, this class.)
- Is it in the package object of the current package?
- Is it being imported, either through a wildcard import (`import MyImplicits._`) or directly (`import MyImplicits.stringToBooleanConverter`)?

---

## Phase 2: Companion Objects

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

## Phase 2: Companion Objects

### Implicit Values

Is it in the companion object of a type involved in the needed type?

```scala
val n: read[Int](s)
val isP = read[Boolean](s)

object Read {
  implicit val readInt: Read[Int] = ???
  implicit val readboolean: Read[Boolean] = ???
}
```

---

## Recap

- The search runs in two phases: immediate (local definitions, direct imports), then the implicit scope (companion objects of "relevant" types).
- If there are multiple valid candidates found in a given phase, the implicit is ambiguous and compilation fails.
- "Relevant" types: the type, any type parameters, and (for conversions) the expected type.

---
template: title

# Uses and Abuses

---
layout: false

# Late Trait Implementation

Mix in a trait after implementation.

```scala
case class Foo(n: Int)

implicit class OrderedFoo(val foo: Foo) extends AnyVal with Ordered[Foo] {
    def compare(that: Foo) = foo.n - that.n
}

Foo(1) < Foo(2)
```

???

- Not a pattern I've seen much, if at all.
- But interestingly, Martin says this is the historical reason why implicits were originally added to the language.

---

# Typeclasses

```scala
trait Read[A] {
    def read(s: String): A
}
```
--
```scala
def read[A](s: String)(implicit ra: Read[A]): A

implicit val stringRead: Read[String] = ???

implicit def seqRead[A]: Read[Seq[A]] = ???

implicit def eitherRead[A, B](implicit ra: Read[A],
                                       rb: Read[B]): Read[Either[A, B]] = ???

implicit def mapRead[A, B](implicit ra: Read[A], 
                                    rb: Read[B]): Read[Map[A, B]] = ???

// Let the compiler compute the reads
val myComplexFoo = read[Map[String, Seq[Either[String, Foo]]](s)
```

---

# Extension Methods

- Usually provided by implicit classes
- Particularly nice for typeclasses; enables an object-oriented feeling API for methods provided through typeclasses.

```scala
trait Functor[F[_]] {
    def fmap[A, B](fa: F[A], f: A => B): F[B]
}

implicit class FunctorOps[F[_], A](fa: F[A])(implicit functor: Functor[F]) {
    // Name `fmap` chosen to avoid conflicting with regular `map`.
    def fmap[B](f: A => B): F[B] = functor.map(fa, f)
}

implicit val optionFunctor: Functor[Option] = ???

Option(2).fmap(_.toString)
    // Desugars to
new FunctorOps(Option(2))(optionFunctor).fmap(_.toString)
```

---

# Theorem Proving

```scala
class List[A] {
  def flatten[B](implicit ev A =:= List[B]): List[B] = ???
}

trait =:=[A, B]
implicit def isEq[A]: A =:= A = new =:=[A, A] {}
```

- `A =:= List[B]` is a theorem.
- `isEq[A]` is a program for proving two types equal.

???

Curry Howard isomorphism. We want to prove a thing, so we give the compiler an understanding of what we want to prove 

---
template: title

# Working with Implicits

---
layout: true
# Best Practices
---

- Always wrap your implicit values in types unique to their usage
???
Easy to end up with conflicts on implicit conversions. Remember you only get one implicit value of each type in a given scope, so you need a way to differentiate them.
--

- Always be explicit about your implicit values' types.
???
Omitting the type can slow down compilation, as well as lead to surprises if a type is wider or narrower than you expected it to be.

---

- Use the "low priority implicits" trick to allow people to override your defaults when necessary.

```scala
trait Reads[A] { /*...*/ }

object Reads extends LowPriorityReadsImplicits {

}

trait LowPriorityReadsImplicits {
    implicit val readsInt: Reads[Int] = ???
    implicit def readsSeq[A: Reads] = ???
}
```

---

- Be conservative with implicit conversions

```scala
implicit val suits = List("clubs", "spades", "hearts", "diamonds")

def shout(s: String) = s"${s.toUpperCase}!!!"

val loud42 = shout(42)
```

What is `loud42`?

--

```
java.lang.IndexOutOfBoundsException: 42
  at scala.collection.LinearSeqOptimized.apply(LinearSeqOptimized.scala:63)
  at scala.collection.LinearSeqOptimized.apply$(LinearSeqOptimized.scala:61)
  at scala.collection.immutable.List.apply(List.scala:86)
  ... 29 elided
```

---

## Puzzling Conversions

```scala
implicit val suits = List("clubs", "spades", "hearts", "diamonds")

def shout(s: String) = s"${s.toUpperCase}!!!"

val loud42 = shout(42)

    // Becomes

val loud42 = shout(suits(42))
```

---
layout: false

# Debugging

If you use IntelliJ, it's your best tool for puzzling out issues with implicits.

- ^⬆P ("Type info") - Check the type of an expression
    - Useful to check if a type matches what you think it is when it isn't converting.
- ⌘⬆P ("Implicit parameters") - Show implicit parameter
    - Useful to see which implicit parameters are missing and (sometimes) why.
- ^Q ("Implicit conversion") - Show implicit conversions that are in scope
    - Useful to see if an implicit conversion is unexpectedly being applied (or not being applied).

???

- Shortcuts are for MacOS X 10.5+ keybindings. The names there are what to search for in the find-anything menu, in case your keybindings are different.
- Use `implicitly[]` to narrow down possibilities
- Use explicit types to try to figure out weird implicit conversion applications and mis-applications.

---
template: title

# Implicit Functions

---
layout: true

# Implicit Functions
---

Recall that this compiles, but doesn't really use implicit parameters.

```scala
def f: A => B = { implicit a => ??? }
```

--

Proposed change: Let implicitness be part of the type.

```scala
def f: implicit A => B = { implicit a => ??? }
```

---

Implicits functions desugar into a new family of ImplicitFunction types, analogous to the existing Function types.

```scala
trait ImplicitFunction1[-T, +R] extends Function1[T, R] {
    def apply(implicit t: T): R
}

// These are equivalent
implicit A => B =:= ImplicitFunction1[A, B]

// The same as
A => B =:= Function1[A, B]
```

---

## The Rules

- Implicit functions pick up their parameters the same as regular implicit methods.
- If the expected type of something is an implicit function, it is expanded to become an implicit function.

```scala
val f: implicit A => B = b
    // Expands to
val f: implicit A => B = implicit (_: A) => b
```

This enables us to write code that depends on context without needing to know about the context itself.

---

```scala
def add5(fn: Future[Int])(implicit ec: ExecutionContext): Future[Int] =
  fn.map(_ + 5)
```
--
```scala
type Execution[A] = implicit ExecutionContext => Future[A]

def add5(fn: Future[Int]): Execution[Int] = fn.map(_ + 5)
```

---

- Could imagine other constructions being passed through this mechanism, like a DB connection or a security context.

```scala
type Secured[A] = implicit SecurityContext => A

// Define a getter to easily get the context from inside an implicit function
// block.
def securityContext(implicit ctx: SecurityContext): SecurityContext = ctx

// Can modify the context if necessary.
def addRole(role: Role)(f: Secured[A]) = {
    val updatedContext = securityContext.addRole(role)
    f(updatedContext)
}

withRole(admin) {
    /* do some stuff */
}
```

---

Similar to the Reader monad.

```scala
trait Reader[A, B] {
    def run(a: A): B
}

object Reader {
    def apply[A, B](f: A => B): Reader[A] 
}
```

- Reader enforces sequencing, since it's a monad.
- Odersky estimates implicit functions to do the same job, but 7x faster.

---

## Mutable Builders

```scala
def table(init: implicit Table => Unit): Table = {
    implicit val t = new Table
    init
    t
}

def row(init: implicit Row => Unit)(implicit t: Table): Row = {
    implicit val r = new Row
    init
    t.add(r)
}

def cell(str: String)(implicit r: Row): Cell = r.add(new Cell(str))
```

---

## Mutable Builders

```scala
table {
    row {
        cell("top left")
        cell("top right")
    }

    row {
        cell("bottom left")
        cell("bottom right")
    }
}
```

---
template: title

# Thanks!

## Questions?

---
layout: false
# References

- Odersky, "Implicit Function Types", [scala-lang.org](https://www.scala-lang.org/blog/2016/12/07/implicit-function-types.html)
- Odersky, et al., "Scala Language Specification, Version 2.12", [scala-lang.org](http://www.scala-lang.org/files/archive/spec/2.12/)
- Odersky, "What to Leave Implicit", Scala Days Chicago 2017, [YouTube](https://youtu.be/Oij5V7LQJsA)