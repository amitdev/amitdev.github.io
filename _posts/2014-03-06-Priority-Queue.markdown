---
layout: post
title: Purely functional Priority Queue
category: Coding
tags: scala functional-programming
year: 2014
month: 03
day: 06
published: true
summary: Implementing a purely functional priority queue data structure.
---

Introduction
------------

This is part of a series of articles on implementing various functional data structures in Scala. See other articles in this series:

- [Functional Stacks](http://amitdev.github.io/coding/2013/12/31/Functional-Stack/)
- [Functional Sets](http://amitdev.github.io/coding/2014/01/20/Functional-Set/)

In this part we will implement a basic Priority Queue in Scala. It will support the following operations efficiently:

```scala
abstract class PriorityQueue[A](implicit val ord: Ordering[A]) {
  // Add an item
  def +(x: A) : PriorityQueue[A]
  // Find min item
  def findMin: A
  // New Priority queue with min item deleted
  def deleteMin(): PriorityQueue[A]
  // Merges two PriorityQueue together
  def meld(that: PriorityQueue[A]) : PriorityQueue[A]
}
```

Basics
------

There are [different](http://en.wikipedia.org/wiki/Priority_queue#Usual_implementation) ways to implement a Priority Queue. We will use [Binomial Heap](http://en.wikipedia.org/wiki/Binomial_heap). As usual we follow the approach from Okasaki. See [this paper](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.48.973) for details about implementing efficient and purely functional priority queues. We start with the basic implementation which is simple to understand. If you know how Binomial heaps work, skip to the [next](#implementation) section. Else read on.

A Binomial heap is a forest of Binomial trees. A binomial tree of rank 0 has just a single node and of rank k (k > 0) has binomial trees of rank (0, 1, .. k-1) as its children. So it looks like (image from wikipedia):

<img src="http://upload.wikimedia.org/wikipedia/commons/c/cf/Binomial_Trees.svg" style="width: 60%; height: 60%"/>

Another way to think about binomial tree is: to get binomial tree of rank k, take binomial tree of rank k-1 and insert another tree of rank k-1 as a child to the first tree's root.

<img src="/img/bt.svg" style="width: 60%; height: 60%"/>

In any case, a Binomial heap of n elements would contain only trees of the above form and it will correspond to (k<sub>1</sub>, k<sub>2</sub>, ..k<sub>j</sub>) where k<sub>1</sub>k<sub>2</sub>..k<sub>j</sub> is the binary representation of n. So a heap of 5 (101) elements would have trees of rank 0 and 2. This means merging two binomial heap is similar to adding two binary numbers etc.

Ok, but how can we use a Binomial heap as a Priority Queue? Let's walk through an example and add the following items: (3, 5, 1, 2, 13, 15, 11, 12) in this order and see what we get:


![drawing](/img/bh3.svg)

So you get the picture. Now to find the min element, we just need to walk through the roots of trees (`log(n)` items) and return the min. Adding an element is similar to adding 1 to a binary number and deletion is similar. Merging (aka melding) is like adding two binary numbers. In all operations all we need to make sure is that the root is properly selected (since that should always have the minimum).

<a name="implementation"></a>
## Implementation

Now that the idea is clear, let's start writing some code. We have a concrete class ``BinomialQueue`` to represent the queue and a ``Node`` class to represent individual trees. Let's look at the ``Node`` class first:

```scala
case class Node[A](data: A, rank: Int = 0, children: List[Node[A]] = Nil)
                          (implicit val ord: Ordering[A]) extends Ordered[Node[A]] {

  def link(other: Node[A]) =
    if (ord.compare(data, other.data) < 0) Node(data, rank+1, other :: children)
    else Node(other.data, other.rank+1, this :: other.children)

  override def compare(that: Node[A]): Int = ord.compare(data, that.data)
}
```

For simplicity, every ``Node`` stores its rank and have data and list of children. It is ordered based on the data (which needs to have an implicit ``Ordering``). Other than this we have a ``link`` method which links two ``Node`` together keeping the min element as new root. For example, in the above figure, when we insert `2`, we link `[1,2]` and `[3,5]` together and `1` gets to be the root.

Now we can define the ``BinomialQueue`` class:

```scala
private final case class BinomialQueue[A] (nodes: List[Node[A]])
                        (implicit override val ord: Ordering[A])
                        extends PriorityQueue[A] {
}
```

Operations
----------

#### Insert

```scala
def +(x: A) : BinomialQueue[A] = BinomialQueue(insertNode(Node(x), nodes))

private def insertNode[T](n: Node[T], lst: List[Node[T]]) : List[Node[T]] = lst match {
  case Nil => List(n)
  case x :: xs =>
    if (n.rank < x.rank) n :: x :: xs
    else insertNode(x.link(n), xs)
}
```

We have 3 case to handle. Inserting to:

- Empty heap : Create a new List having a rank-0 Node (Eg. inserting 3 above)
- Heap without a rank-0 node: Just add a rank-0 Node to our list (Eg. inserting 1, 13 etc above)
- Heap with a rank-0 Node: Now we need to merge and recursively insert merged Node (Eg. inserting 5, 2 etc)

#### Find Minimum

Just returns the ``Node`` with min element.

```scala
def findMin: A = nodes.min.data
```

#### Merge

```scala
def meld(that: PriorityQueue[A]) : PriorityQueue[A] =
  BinomialQueue(meldLists(this.nodes, that.nodes))

private def meldLists[T](q1: List[Node[T]], q2: List[Node[T]]) : List[Node[T]] = (q1, q2) match {
  case (Nil, q) => q
  case (q, Nil) => q
  case (x :: xs, y :: ys) => if (x.rank < y.rank) x :: meldLists(xs, y :: ys)
  else if (x.rank > y.rank) y :: meldLists(x :: xs, ys)
  else insertNode(x.link(y), meldLists(xs, ys))
}
```

Logic is similar to insert: If a particular ranked node does not exist on one list we can just copy it to output. Same ranked Nodes are merged via linking them together.

### Delete Minimum

```scala
def deleteMin(): PriorityQueue[A] = {
  val minNode = nodes.min
  BinomialQueue(meldLists(nodes.filter(_ != minNode), minNode.children.reverse))
}
```

Finds the min Node and removes it. The children of a Binary tree conveniently forms a Binary heap in reversed order, so we just merge that to other Nodes.


Adapting to Scala Collections
-----------------------------

We are done and have a working Priority Queue implementation (we could make some operations more efficient by using Skew binary heaps, but thats for a later time). However, the other common operations on collections like iterating, map, filter etc are missing. Scala has a rich collections library and it is not difficult to reuse most of it for a new collection object. See this [excellent](http://docs.scala-lang.org/overviews/core/architecture-of-scala-collections.html) guide to understand the design of Scala collections and how to adapt your collection to use it.

In a nutshell we need to do two things: Provide a way to iterate over the items (``Traversable`` or ``Iterable``) and a way to build a new collection (``Builder``). There are also higher level of abstractions that make things easier. So we change the ``PriorityQueue`` definition as follows:

```scala
abstract class PriorityQueue[A](implicit val ord: Ordering[A])
  extends Iterable[A]
  with GenericOrderedTraversableTemplate[A, PriorityQueue]
  with IterableLike[A, PriorityQueue[A]] {
```

That's quite a mouthfull, so let's take it one by one. [Iterable](http://docs.scala-lang.org/overviews/collections/trait-iterable.html) just defines an ``iterator`` method which yields elements of the collection in some order and an ``isEmpty`` method. We can implement it as follows:

```scala
override def isEmpty: Boolean = nodes.isEmpty

//Inside BinomialHeap
override def iterator: Iterator[A] = nodes.flatMap(_.toList).iterator

//Inside Node:
def toList : List[A] = data :: children.flatMap(_.toList)
```

``IterableLike`` is for ensuring the same-result-type principle. [GenericOrderedTraversableTemplate](http://www.scala-lang.org/api/2.10.3/index.html#scala.collection.generic.GenericOrderedTraversableTemplate) is an easy way to define the builder using a companion object for ordered collections. It has following methods to be implemented:

```scala
override def orderedCompanion: GenericOrderedCompanion[PriorityQueue] = PriorityQueue
override protected[this] def newBuilder: mutable.Builder[A, PriorityQueue[A]] =
                                                             PriorityQueue.newBuilder
```

Basically we just need to tell which is the companion object where the builder is specified. Finally we need to define the companion object which defines how to build a ``BinomialQueue`` from other collections:

```scala
object PriorityQueue extends OrderedTraversableFactory[PriorityQueue] {
  override def newBuilder[A](implicit ord: Ordering[A]):
                                 mutable.Builder[A, PriorityQueue[A]] =
    new ArrayBuffer[A] mapResult { xs =>
      xs.foldLeft(BinomialQueue[A](Nil))((t, x) => t + x)
    }

  implicit def canBuildFrom[A](implicit ord: Ordering[A]):
   CanBuildFrom[Coll, A, PriorityQueue[A]] = new GenericCanBuildFrom[A]
}
```

Now we can use all the power of Scala collections.

```scala
val pq = PriorityQueue(1,2,3,4,5,6)        // BinomialQueue(5, 6, 1, 3, 4, 2)
pq map (_ * 2) filter (_ < 6)              // BinomialQueue(2, 4)
val sq = PriorityQueue("a", "aa", "aaaaa") // BinomialQueue(aaaaa, a, aa)
sq map (_.length) reduce (_*_)             // 10

```
Isn't that cool?

You can find the complete code [here](https://github.com/amitdev/functional-ds/blob/master/src/main/scala/ds/PriorityQueue.scala). The Binomial heap images were generated using [this code](https://github.com/amitdev/functional-ds/blob/master/src/main/scala/ds/PriorityQueueUtil.scala).
That's all for now, see you later!
