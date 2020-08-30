---
layout: post
title: Optics to modify State
date: 2020-08-19 01:22
summary: Coping with one of the core concepts in Functional Programming.
categories: ['theory', 'scala']
tags: ['scala', 'cats', 'monocle']
---
Let's talk about the immutable state.

You, probably, have heard already a lot about the immutable state in functional programming. I'm not gonna try to convince you again
on this. In this article, I want to show you how you can simplify your life, meanwhile still having all the benefits of having no state.

### Immutability is cumbersome
<sup>*If you know what're Optics and State, you can skip to the [main part](#optics-with-state).*</sup>

One of the problems with immutability that you must write a lot of boilerplate. Especially when you have deeply nested structures.

```scala
case class Street(number: Int, name: String)
case class Address(city: String, country: String, street: Street)
case class Person(name: String, address: Address)

val p = Person("John", Address("New York", "USA", Street(1, "Park Ave")))
```

Now to change a person's street number, you'd do:

```scala
p.copy(
  address = p.address.copy(
    steet = p.address.street.copy(
      number = 2
    )
  )
)
```

Also there is another problem with immutable objects - you must capture each intermediate state updates in different variables:
```scala
case class Random(seed: Long) {
  def next = Random(seed * 134775813 + 1)
}

def randomLong(r: Random): (Random, Long) = (r.next, r.seed)
def randomInt(r: Random): (Random, Int) = (r.next, (r.seed >> 32).toInt)
def randomBoolean(r: Random): (Random, Boolean) = (r.next, r.seed % 2 == 0)

case class Entry(l: Long, i: Int, b: Boolean)

def randomEntry(r: Random): (Random, Entry) = {
  val (r1, l) = randomLong(r)
  val (r2, i) = randomInt(r1)
  val (r3, b) = randomBoolean(r2)
  (r3, Entry(l, i, b))
}
```

Not only this is inconvenient, but also it's a source of possible mistake, when you misspell intermediate state variable.

## Solutions

Fortunately, both this problems are solved in Functional Programming.<br>
**Optics** cope with the deeply nested data structures, whereas the intermediate state updates can be captured by the **State** monad.

### Optics
Optics are a family of classes which allow you to access and modify data, potentially very deep within a structure. <br>
[Monocle](https://www.optics.dev/Monocle "Monocle - Optics for Scala") is a Scala library which gives you those abstractions.
**Lens** and **Optional** are most notable Optics. Former consists of `get` and `set` functions:

```scala
abstract class Lens[Structure, A] {
  def get(s: Structure): A
  def set(a: A)(s: Structure): Structure
}
```

It allows you to 'look into' the `A` part of the `Structure`.

The Optional is very similar to Lens, but it indicates that the `A` part may be missing in the `Structure`:
```scala
abstract class Optional[Structure, A] {
  def getOptional(s: Structure): Option[A]
  def set(a: A)(s: Structure): Structure
}
```

The cool part about `Lens` and `Optional` is that they compose:
```scala
case class Inner(value: String)
case class Outer(inner: Inner)
val valueLens: Lens[Inner, String] = Lens[Inner, String](_.value)(newV => _.copy(value = newV))
val innerLens: Lens[Outer, Inner] = GenLens[Outer, Inner](_.inner) // shortcut

val deepLens: Lens[Outer, String] = innerLens composeLens valueLens

val structure = Outer(Inner("value"))

deepLens.set("new-value")(structure)
//> Outer(Inner(new-value))
```

Let's revise the initial example with the Person:
```scala
case class Street(number: Int, name: String)
case class Address(city: String, country: String, street: Street)
case class Person(name: String, address: Address)

val numberLens: Lens[Street, Int] = GenLens[Street, Int](_.number)
val streetLens: Lens[Address, Street] = GenLens[Address, Street](_.street)
val addressLens: Lens[Person, Address] = GenLens[Person, Address](_.address)

val personStreetNumber: Lens[Person, Int] =
 addressLens composeLens streetLens composeLens numberLens

val p: Person = ???

val res = personStreetNumber.set(2)(p)
assert(res.address.street.number == 2)
```

### State

State monad is a data structure, which is basically a wrapper for a function `S => (S, A)`. Because it's a monad, it has the `flatMap` function,
the semantics in this case - it passes intermediate state though the call chain.

```scala
case class Random(seed: Long) {
  def next = Random(seed * 134775813 + 1)
}

val randomLong: State[Random, Long] = State { r => (r.next, r.seed) }
val randomInt: State[Random, Int] = randomLong.map(l => (l.seed >> 32).toInt }
val randomBoolean: State[Random, Boolean] = randomLong.map(_ % 2 == 0)

case class Entry(l: Long, i: Int, b: Boolean)

val randomEntry: State[Random, Entry] = {
  for {
    l <- randomLong
    i <- randomInt
    b <- randomBoolean
  } yield Entry(l, i, b)
}
```

You can get the `State` monad by using [Cats](https://typelevel.org/cats/ "Cats") library.

## Optics with State

The neat thing, that you can use both of them to make your code even cleaner. Say you have an immutable state and you want to modify only a part of it:
```scala
case class Inner(string: String, boolean: Boolean)
case class Outer(map: Map[String, Inner])

def modifyBoolean(key: String, newBool: Boolean): State[Outer, Unit] =
  State.modify[Outer] { o =>
    val inner: Option[Inner] = o.map.get(key)
    inner.fold(o) { i =>
      val newMap = o.map.updated(key, i.copy(boolean = newBool))
      o.copy(map = newMap)
    }
  }
```

If you have Optional, it'd look like:

```scala
import monocle.function.Index._

val mapLens: Lens[Outer, Map[String, Inner]] = GenLens[Outer](_.map)
val boolLens: Lens[Inner, Boolean] = GenLens[Inner](_.boolean)
def optional(key: String): Optional[Outer, Inner] =
  mapLens composeOptional index(key) composeLens boolLens

def modifyBoolean(key: String, newBool: Boolean): State[Outer, Unit] =
  State.modify[Outer] { o =>
    optional(key).set(o)(newBool)
  }

def getBoolean(key: String): State[Outer, Boolean] =
  State.inspect[Outer, Option[Boolean]](optional(key).getOption).map(_.exists(identity))
```

You can even introduce extension for Optics, as a shortcut for State modification:

```scala
import cats.Applicative
import cats.data.{State, StateT}
import cats.instances.all._
import cats.syntax.all._
import monocle.Lens

implicit class RichLens[S, A](lens: Lens[S, A]) {
  def applyS[B](f: A => (A, B)): State[S, B] = State[S, B] { s =>
    val ab = f.compose(lens.get)(s)
    ab.leftMap(lens.set(_)(s))
  }

  def modifyS(f: A => A): State[S, Unit] =
    State.modify[S](lens.modify(f))

  def inspectS[B](f: A => B): State[S, B] =
    State.inspect(f.compose(lens.get))

  def setS(a: A): State[S, Unit] =
    State.modify(lens.set(a))

  // With Effects
  
  def applySt[F[_]: Applicative, B](f: A => F[(A, B)]): StateT[F, S, B] = StateT[F, S, B] { s =>
    val fab = f.compose(lens.get)(s)
    fab.map(_.leftMap(lens.set(_)(s)))
  }
  
  def modifySt[F[_]: Applicative, B](f: A => F[A]): StateT[F, S, Unit] =
    StateT.modifyF(lens.modifyF(f))
  
  def inspectSt[F[_]: Applicative, B](f: A => F[B]): StateT[F, S, B] =
    StateT.inspectF(f.compose(lens.get))
}

// similar for RichOptional
```

So now you can have a compact version of the previous example:

```scala
def optional(key: String): Optional[Outer, Inner] = ???

def modifyBoolean(key: String, newBool: Boolean): State[Outer, Unit] =
  optional(key).setS(newBool)
  
def getBoolean(key: String): State[Outer, Boolean] =
  optional(key).inspectS(_.exists(identity))
```

Now you can use Optics to modify a part of a state and create a State monad for that modification!

### Summary

If you ever decide to have immutable state in your application, don't hezitate to use **Optics** and **State** monad.
Optics are useful for accessing and modifying deeply nested data structures. The State monad is useful for passing
through intermediate state.

And now you also know how to use both and achieve an even more concise solution.
