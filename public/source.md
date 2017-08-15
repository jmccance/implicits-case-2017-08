name: title
layout: true
class: center, middle, section
---
template: title

# Implicits

How to Say a Lot Without Saying Anything At All

Joel McCance  
[@jmccance](http://twitter.com/jmccance)  
17 August 2017

---
layout:false

# Agenda

1. Implicits 101
2. Deep dive
3. Implicit functions

---
class: middle

> "If there's one feature that makes Scala 'Scala', I would pick implicits. There's hardly an API without them. They enable advanced and elegant architectural designs. They are also misused way too often."  
  <cite><strong>Martin Odersky</strong>, Scala Days Chicago Keynote, 2017</cite>

---
template: title

# Implicits 101

---
layout: true

# Implicit Parameters

---

## Motivation

```scala
def f(asdf: String, context: Context): Unit
def g(zxcv: Bar, context: Context): Foo
```

---

## I

---
layout: false

# References

- Odersky, "Implicit Function Types", [Scala Blog](https://www.scala-lang.org/blog/2016/12/07/implicit-function-types.html)
- Odersky, "What to Leave Implicit", Scala Days Chicago 2017, [YouTube](https://youtu.be/Oij5V7LQJsA)