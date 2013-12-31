---
layout: post
title: A basic, immutable Stack in Scala
category: Coding
tags: scala functional-programming
year: 2013
month: 12
day: 31
published: true
summary: Implementing an object-functional Stack data structure
---

Introduction
------------

[Scala](http://scala-lang.org/) was one language that I played around with many years back (and [blogged](http://memuser.blogspot.in/search/label/Scala) about then). Recently, I took the excellent [course](https://www.coursera.org/course/reactive) on reactive programming which rekindled my interest in Scala and I decided to brush up my knowledge on it as well as functional programming. I also happen to be reading what seems to be the best resource on functional data structures - Chris Okasaki's [Purely Functional Data Structures](http://www.amazon.com/Purely-Functional-Structures-Chris-Okasaki/dp/0521663504). This blog is an introduction on how to implement some of those data structures in Scala. Since Scala is a general purpose programming language with an objective to fuse object oriented and functional programming, our implementation will follow that approach. It is also possible to do pure functional programming in Scala without bringing in object orientation, but that's not our focus here. So I'll call this implementation object-functional versus purely functional, but the properties are the same, the differences are only in the way code is organized.


Stack
-----

We start with implementation of a really simple data structure. It is a container which supports the following operation:

- ``cons``: adds a new item to the container
- ``head`` : get last inserted item
- ``tail`` : get rest of the items
- ``isEmpty``: the container is empty or not

You can think of it as a [cons list](http://en.wikipedia.org/wiki/Cons) or Stack as Okasaki calls it. Before thinking about implementation, we can capture the behavior in a Trait like so:

```scala
trait Stack[T] {
  def isEmpty: Boolean
  def cons(t: T): Stack[T]
  def head: T
  def tail: Stack[T]
}
```

Digression: Variance
--------------------

In Scala, the type parameters are invariant by default, which means if you have a ``Stack[Shape]``, you cannot add a ``Circle`` to it though ``Circle`` is a subclass of ``Shape``. For immutable collections it is better to have this behavior, so we need to make it covariant which can be easily done by adding a ``+`` to the type parameter. When you do that, ``Stack[Shape]`` will behave like a superclass of ``Stack[Circle]``. But this also means that the following is possible:

```scala
val onlyCircles: Stack[Circle] = ...
val shapes: Stack[Shape] = onlyCircles
shapes.cons(new Square)
```

This is clearly a problem so Scala compiler will prevent you from creating such an ``cons`` function. In the example above, ``cons`` should ensure that it only accepts a ``Circle`` or its subclass. This can be specified by constraining its type to ``[U >: T]`` where ``T`` is the type parameter for ``Stack``. This is the intuition behind variance (we skipped contravariance) and if you are interested, read more [here](http://twitter.github.io/scala_school/type-basics.html#variance).

Armed with the idea of variance, we can make our ``Stack`` trait more general:

```scala
trait Stack[+T] {
  def isEmpty: Boolean
  def cons[U >: T](t: U): Stack[U]
  def head: T
  def tail: Stack[T]
}
```

Implementation
--------------

One way to implement ``Stack`` is to consider two cases. A ``Stack`` could be empty or it could have a head and tail. Further, empty can be a singleton and we introduce an abstract class ``CList`` to represent this particular implementation:

```scala
sealed abstract class CList[+T] extends Stack[T] {
  def cons[U >: T](t: U): CList[U] = new Cons(t, this)
}

case object Empty extends CList[Nothing] {
  def isEmpty: Boolean = true
  def head: Nothing = throw new NoSuchElementException()
  def tail: Nothing = throw new NoSuchElementException()
}

case class Cons[+T](hd: T, tl: CList[T]) extends CList[T] {
  def isEmpty: Boolean = false
  def head: T = hd
  def tail: CList[T] = tl
}
```

Given this, one could create a Stack like:

```scala
val stack = Cons(1, Cons(2, Empty)) // head = 1, tail = [2]
```

Persistent
----------

An important property of functional data structures is that they are immutable, any change on them would give us a new copy. This may not be as bad as it sounds since the structure can be often shared. For instance, if we have a list: ``1 -> 2 -> 3`` and we append 0 to it, we get a new list: ``0 -> 1 -> 2 -> 3``, but the tail of second list is just a pointer to first list, so no extra space is required (except for an extra pointer). If we insert an element in between, the elements till that needs to be copied but rest can be shared. If we append two lists, ``xs`` and ``ys``, all elements in ``xs`` needs to be copied but ``ys`` can be shared etc. There are ways to optimize this to minimize copying.

But keeping things immutable has many [advantages](http://stackoverflow.com/questions/4399837/what-is-the-benefit-of-purely-functional-data-structure). One particular benefit is even after updates the previous 'version' of the data structure is available since we don't modify anything. For this reason, functional data structures are also called persistent.

Other operations
----------------

The implementation we have so far is trivial and we can add other operations:

- ``update``: inserts an element at a given index.
- ``++``: catenates two Stacks together

The implementation for this is straightforward. We just need to handle the two cases. Let's also add a ``toList`` function to convert this to Scala ``List`` for debugging.

```scala
sealed abstract class CList[+T] extends Stack[T] {
  def ++[U >: T](ys: CList[U]): CList[U]
  def update[U >: T](t: U, i: Int): CList[U]
  def toList: List[T]
}

case object Empty extends CList[Nothing] {
  def ++[U](ys: CList[U]): CList[U] = ys
  def update[U](t: U, i: Int) = throw new IndexOutOfBoundsException()
  def toList = Nil
}

case class Cons[+T](hd: T, tl: CList[T]) extends CList[T] {
  def ++[U >: T](ys: CList[U]): CList[U] = Cons(head, tail ++ ys)
  def update[U >: T](t: U, i: Int): CList[U] = if (i == 0) Cons(t, this) else Cons(head, tail.update(t, i-1))
  def toList = head :: tail.toList
}
```

Note that the ``update`` and ``++`` operations return a new Stack which potentially shares the structure of old Stack as we discussed before.

Digression: Testing
-------------------

Now that we have written some code, how do we test it? We need to have some test data and unit tests for testing Stack. Also we need to test for ``Empty`` and ``Cons`` case separately. Instead of writing specific test cases, let's use [Scalacheck](http://www.scalacheck.org/). Scalacheck basically allows us to assert properties or invariants about the data structure and it takes care of generating random data for us.

Since we have a new data structure, which Scalacheck does not know about, we need to describe how to generate data. One way to do it is:

```scala
implicit def arbStack[T](implicit a: Arbitrary[T]): Arbitrary[CList[T]] = Arbitrary {
  def genStack: Gen[CList[T]] = oneOf(Empty, for {
    v <- arbitrary[T]
    s <- genStack
  } yield s.cons(v))
  genStack
}
```

It might look a bit complicated, but all it is doing is describing how we can construct arbitrary Stacks. Once this is done, we can start writing our tests by just describing properties:


```scala
test("Stack head") {
  check((s: CList[Int], x: Int) => s.cons(x).head == x)
}

test("Stack tail") {
  check((s: CList[Int], x: Int) => s.cons(x).tail == s)
}

test("Stack append") {
  check((xs: CList[Int], ys: CList[Int]) => size(xs ++ ys) == size(xs) + size(ys))
}

test("Stack update") {
  check((xs: CList[Int], x: Int, i: Int) => {
    if (i >= 0 && i < size(xs)) size(xs.update(x, i)) == size(xs)+1
    else try {
      xs.update(x, i)
      false
    } catch {
      case _:IndexOutOfBoundsException => true
    }
  })
}
```

Note that we did not provide any sample ``CList``. ScalaCheck will do that for us by testing with various randomly generated ``CList``.

Exercise
--------

Implementing a ``suffixes`` operation is provided as an exercise - given a ``Stack`` generate all its suffices. So suffixes of ``[1,2,3]`` should be ``[[1,2,3], [2,3], [3], []]``. It has to be generating in decreasing order of size. Given our classes, this can be easily done:

```scala
sealed abstract class CList[+T] extends Stack[T] {
  def suffixes: CList[CList[T]]
}

case object Empty extends CList[Nothing] {
  def suffixes = Cons(Empty, Empty)
}

case class Cons[+T](hd: T, tl: CList[T]) extends CList[T] {
  def suffixes: CList[CList[T]] = Cons(this, tail.suffixes)
```

and a simple test which passes:

```scala
test("Stack suffixes") {
  check((xs: CList[Int]) => {
    val suffixes = xs.suffixes
    size(suffixes) == size(xs) + 1
  })
}
```

That is all for a basic implementation of ``Stack``. See the complete code for [Stack](https://github.com/amitdev/functional-ds/blob/master/src/main/scala/ds/Stack.scala) and the [test](https://github.com/amitdev/functional-ds/blob/master/src/test/scala/StackTest.scala). We could add many operations like ``map``, ``filter`` etc in a similar way. If you are interested, take a look a Scala's implementation of [List](https://github.com/scala/scala/blob/master/src/library/scala/collection/immutable/List.scala) which is more general and feature rich.

