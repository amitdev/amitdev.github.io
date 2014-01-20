---
layout: post
title: A basic, immutable Set in Scala
category: Coding
tags: scala functional-programming
year: 2014
month: 01
day: 20
published: true
summary: Implementing an object-functional Set data structure
---

Introduction
------------

In the [last article](http://amitdev.github.io/coding/2013/12/31/Functional-Stack.html) we saw one way of implementing an object-functional, immutable Stack data structure. Now let's see how to implement a Set data structure. A Set is a container which supports the following operations efficiently:

- ``insert``: adds a new item to the Set
- ``delete``: removes given item from the Set
- ``member``: check if a given item is in the Set

As we did before with the Stack, we can define this behavior in a Trait like the following:

```scala
trait Set[+T] {
  def insert[U >: T](item: U) : Set[U]
  def delete[U >: T](item: U) : Set[U]
  def member[U >: T](item: U) : Boolean
}
```

Ordering
--------

We are planning to implement the Set using [Binary Search Trees](http://en.wikipedia.org/wiki/Binary_search_tree) (BST) which means elements in the Set needs to be comparable (or can be ordered). One way to do it is by constraining the type ``T`` to be a subclass of ``Ordered[T]``. But a more lenient approach is to allow the caller to use any type as long as it can be [implicitly](http://www.artima.com/pins1ed/implicit-conversions-and-parameters.html) converted to an ``Ordered[T]``. Scala provides syntactic sugar for doing this easily called [view bounds](http://twitter.github.io/scala_school/advanced-types.html) using which we can define the type as ``+T <% Ordered[T]``. Since an implicit parameter needs to be passed under the hood, we define ``Set`` as an abstract class. A partial definition will look like the following:


```scala
sealed abstract class Set[+T <% Ordered[T]] {
  def insert[U >: T <% Ordered[U]](x: U) : Set[U]
}
```

There are [plans](https://issues.scala-lang.org/browse/SI-7629) to deprecate view bounds from Scala. We could use the equivalent:

```scala
sealed abstract class Set[+T](implicit t2ord: T => Ordered[T]) {
  def insert[U >: T](x: U)(implicit u2ord : U => Ordered[U]) : Set[U]
}
```

This is a bit verbose since we have to repeat the implicit parameter at multiple places.  A better option is to use [context bounds](http://stackoverflow.com/questions/2982276/what-is-a-context-bound-in-scala). We can also define a type alias for ``T => Ordered[T]`` to simplify things:

```scala
type Orderable[T] = T => Ordered[T]

sealed abstract class Set[+T : Orderable] {
  def insert[U >: T : Orderable](x: U) : Set[U]
  def delete[U >: T : Orderable](x: U) : Set[U]
  def member[U >: T : Orderable](x: U) : Boolean
}
```

Unbalanced BST
--------------

Now that we are done with defining proper types, let's start with the actual implementation. We will first implement an unbalanced version. A BST node can either be a ``Leaf`` which is a sentinel for empty, or a ``Node`` which has a value, left tree and right tree. Once we have this, implementing the operations recursively is trivial since the structure of the tree is also recursive. For instance, to insert a value we need to compare with the root and if the value is less insert on left tree otherwise on right tree etc.

```scala
abstract class BST[+T : Orderable] extends Set[T] {
  def insert[U >: T : Orderable](x: U) : BST[U]
}

object BST {
  class Leaf extends BST[Nothing] {
    def member[U : Orderable](x: U) = false
    def insert[U : Orderable](x: U) : BST[U] = Node(x, Leaf, Leaf)
  }

  class Node[+T : Orderable](y: T, left: BST[T], right: BST[T]) extends BST[T] {
    def member[U >: T : Orderable](x: U) =
                               if (x < y) left.member(x)
                               else if (x > y) right.member(x)
                               else true

    def insert[U >: T : Orderable](x: U) : BST[U] =
                               if (x < y) Node(y, left.insert(x), right)
                               else if (x > y) Node(y, left, right.insert(x))
                               else this
  }
}
```

As is the case with persistent data structures, the operations will result in new copies with potential sharing of structure. For instance, if we insert a value which is less than that in root node, the complete right subtree can be shared etc.

### Deletion

Deleting a node from BST is slightly more [complex](http://en.wikipedia.org/wiki/Binary_search_tree#Deletion). One way is to replace the deleted node with its inorder predecessor. Additionally we need to reconstruct the subtree containing the deleted and predecessor nodes since our data structure is immutable. There are special cases to consider like one or both children being empty which can be handled by the ``Leaf`` object. In the following implementation we use ``delete`` method to locate the node to be deleted and ``deleteCurrent`` to actually replace stuff. It uses a recursive ``fixNodes`` function which finds the predecessor and propagates it back to deleted node rebuilding the tree.

```scala
class Leaf extends BST[Nothing] {
  def delete[U : Orderable](x: U) : BST[U] = this
  protected def deleteCurrent[U : Orderable](t: BST[U]) = t
  protected def fixNodes[U : Orderable] = null
}

class Node[+T : Orderable](y: T, left: BST[T], right: BST[T]) extends BST[T] {
  def delete[U >: T : Orderable](x: U) : BST[U] =
                    if (x < y) Node(y, left.delete(x), right)
                    else if (x > y) Node(y, left, right.delete(x))
                    else left.deleteCurrent(right)

  protected def fixNodes[U >: T : Orderable] : (U, BST[U]) = {
    val r = right.fixNodes
    if (r != null) {
      val (ny, nr) = r
      (ny, Node(y, left, nr))
    } else (y, left)
  }

  protected def deleteCurrent[U >: T : Orderable](t: BST[U]): BST[U] = {
    val (ny, nl) = fixNodes
    Node(ny, nl, t)
  }
}
```

Balanced BST
------------

As you would know, we need a better solution to work efficiently on all inputs. The tree operations take ``O(d)`` time where d is the depth of the tree. If we insert *n* items which are in sorted order, we create a tree with depth *n* which makes the operations linear time. To avoid this, we need to keep the tree balanced so that the depth is roughly *log(n)*. There are multiple ways to do this like AVL trees, Red Black trees(RBT) etc. We will consider a Red black tree implementation here.

Okasaki gives an elegant implementation of RBT in the [book](http://www.amazon.com/Purely-Functional-Structures-Chris-Okasaki/dp/0521663504) (If you don't have a copy see [this](http://www.cs.tufts.edu/~nr/cs257/archive/chris-okasaki/redblack99.ps) paper). As usual, we will adopt the solution to Scala and it is natural to extend from the ``BST``, ``Leaf`` and ``Node`` classes which we already have. We need an additional method ``balancedInsert`` to take care of balancing, so let's create a Trait to capture RBT specific stuff:

```scala
trait RBT[+T] extends BST[T] {
  def balancedInsert[U >: T : Orderable](x: U) : RBT[U]
  def insert[U >: T : Orderable](x: U) : RBT[U]
}
```

RBT is a BST with the following additional properties which guarantees that the tree remains balanced:

1. Every node is either Red or Black. The ``Leaf`` nodes are black by convention.
2. A red node cannot have a red parent
3. Every path from root to leaf should have same no.of black nodes.

``balancedInsert`` makes sure that (2) and (3) above are not violated when inserting an item. We start defining the ``Node`` for RBT as follows:

```scala
sealed abstract class Color
case object Red extends Color
case object Black extends Color

case class Node[+T : Orderable](color: Color, v: T, left: RBT[T], right: RBT[T])
                                extends BST.Node[T](v, left, right) with RBT[T] {

  override def insert[U >: T : Orderable](e: U) : RBT[U] = {
    val Node(_, a1,a2,a3) = balancedInsert(e)
    Node(Black, a1, a2, a3)
  }

}
```

Note that we don't need to define the ``member`` function here. The behavior is same as that of a ``BST`` and since we inherit from ``BST.Node`` we get that for free. We override ``insert`` and make sure we balance things when something is inserted if needed. Finally we maintain the invariant that the root node is always Black.

Let's define the ``Leaf`` for RBT which is simple enough:

```scala
case object Leaf extends BST.Leaf with RBT[Nothing] {
  override def insert[U : Orderable](x: U) = Node(Red, x, Leaf, Leaf)
  def balancedInsert[U : Orderable](x: U)  = insert(x)
}

```

So far we haven't done much in RBT and once we define ``balancedInsert`` we are done! Of course balancing the tree is the heart of ``insert`` operation. As discussed in the paper, we just need to handle balancing (by rotation) in four cases:

<img src="/img/rbt.svg" alt="Balancing Cases" style="width: 60%; height: 60%"/>

This can be declaratively expressed in code, thanks to pattern matching as follows:

```scala
def balancedInsert[U >: T : Orderable](e: U): RBT[U] = {
  def balance(cn: RBT[U]): RBT[U] = cn match {
    case Node(Black, z, Node(Red, y, Node(Red, x, a, b), c), d) =>
      Node(Red, y, Node(Black, x, a, b), Node(Black, z, c, d))
    case Node(Black, z, Node(Red, x, a, Node(Red, y, b, c)), d) =>
      Node(Red, y, Node(Black, x, a, b), Node(Black, z, c, d))
    case Node(Black, x, a, Node(Red, y, b, Node(Red, z, c, d))) =>
      Node(Red, y, Node(Black, x, a, b), Node(Black, z, c, d))
    case Node(Black, x, a, Node(Red, z, Node(Red, y, b, c), d)) =>
      Node(Red, y, Node(Black, x, a, b), Node(Black, z, c, d))
    case c => c
  }

  if (e < v) balance(Node(color, v, left.balancedInsert(e), right))
  else if (e > v) balance(Node(color, v, left, right.balancedInsert(e)))
  else this
}
```

``balance`` does the actual rotation and it corresponds as is to the above diagram. This, in fact, is the heart of the algorithm. Most of the other things we talked about are about how we organize code cleanly, adding flexible type support etc.

Side Note: We didn't tackle deletion of a node here. Implementing the ``delete`` operation is slightly involved for ``RBT``. See [this](http://matt.might.net/articles/red-black-delete/) post to see how it can be done.

Example
-------

Let's see a sample tree construction to see how it works. Assume we insert 1,2,3,4 in this order, we get something like the following:

<img src="/img/s.svg" alt="1" style="width: 80%; height: 80%"/>

We are only discussing the very basic operations of the data structure here. Using this other convenience functions can be easily added. For example, we can add an ``apply`` function to help creating a Set from its arguments as follows:

```scala
def apply[T : Orderable](xs: T*) = xs.foldLeft(Leaf.asInstanceOf[RBT[T]])((t, x) => t.insert(x))
```

Testing
-------

As we did before, tests using [Scalacheck](http://www.scalacheck.org/) can be written once we have a generator of random trees.

```scala
implicit def arbBST[T : Orderable](implicit a: Arbitrary[T]) = Arbitrary {
  for {
    v <- Arbitrary.arbitrary[List[T]]
  } yield BST(v:_*)
}

test("Set Insert") {
  check((s: BST[Int], x: Int) => s.insert(x).member(x))
}

test("Set delete") {
  check((s: BST[Int], x: Int) => !s.delete(x).member(x))
}

```

That's all for now. See the complete code for [Set](https://github.com/amitdev/functional-ds/blob/master/src/main/scala/ds/Set.scala). Scala standard library also provides an implementation of [RBT](https://github.com/scala/scala/blob/master/src/library/scala/collection/immutable/RedBlackTree.scala) in a different style.
