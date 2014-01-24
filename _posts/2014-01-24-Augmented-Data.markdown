---
layout: post
title: Drawing Trees
category: Coding
tags: scala functional-programming
year: 2014
month: 01
day: 24
published: true
summary: Augmenting data and drawing Trees
---

Introduction
------------

In the [last article](http://amitdev.github.io/coding/2014/01/20/Functional-Sets.html) we saw one way of implementing Binary Search Trees and Red Black Trees. This data structure can be used to implement a Set (like we saw before), but there are many other use cases if we can *augment* the data stored on the nodes. One example is a TreeMap which is a key-value store - similar to HashMap but with *log(n)* time guarantee for operations. Other [examples](http://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-046j-introduction-to-algorithms-sma-5503-fall-2005/video-lectures/lecture-11-augmenting-data-structures-dynamic-order-statistics-interval-trees/lec11.pdf) are Interval trees, dynamic order statistics etc.

How to Augment the Data
-----------------------

Recall that a ``Node`` in our RBT looked like the following:

```scala
case class Node[+T : Orderable](color: Color, v: T, left: RBT[T], right: RBT[T])
```

One way to augment the data is to subclass ``Node`` and add relevant data to that class. But there is a simpler approach. Note that in the above declaration, the data stored has an abstract type ``T`` whose only constraint is that it should be ``Orderable``. So we could package all the data we need in ``T`` itself and provide appropriate ordering logic. For example, in case of a TreeMap, ``T`` can be a pair of (K, V) so that ordering is only done on K. A generic ``AugmentedData`` class can be written as follows:

```scala
case class AugmentedData[T : Orderable, U](v: T, data: U)
                              extends Ordered[AugmentedData[T, U]] {
  def compare(that: AugmentedData[T, U]): Int = this.v.compare(that.v)
}
```

So an ``AugmentedData`` would have a value ``v`` which is orderable and additional ``data`` of some type ``U``.

Example - Drawing Trees
-----------------------

Let's look at an example where we augment data. Let's say we need to draw a given RBT (or BST). There are two logical steps to it:

- Identify position of each Node in the picture
- Draw the picture

We separate these two steps because drawing a picture can be done in different ways - as an image, in a terminal etc.

### Step 1 - Finding positions

First we need to Augment the data stored in a ``Node`` to include its position (x,y) as well. Then we need to find and populate the positions.


We could just use the depth of a node in the tree as its vertical position (y-coordinate). Finding a horizontal position (x-coordinate) is slightly tricky and there are many different ways to do it. We will use the simple approach of taking a node's position in the inorder traversal of the tree. By definition, node on the left comes first and will have a lower x coordinate value than its root which will be lesser than its right child etc. This is not the best approach but suffices for our discussion.

We are going to define a function which takes a RBT node and gives back an RBT node which is the root of tree with augmented data. So it will look like:

```scala
def toPos[T : Orderable](node: RBT[T], d: Int = 80, start: Int = 30) : RBT[AugmentedData[T, (Int, Int)]] = ???
```

It has two additional parameters; to adjust the space between nodes: ``d`` and start position: ``start``. To simplify things, we can use a helper function ``calcPos`` which finds the position and propogates it back.

```scala
def toPos[T : Orderable](node: RBT[T], d: Int = 80, start: Int = 30) : RBT[AugmentedData[T, (Int, Int)]] = {
  def calcPos(n: RBT[T], x: Int, y: Int) : (RBT[AugmentedData[T, (Int, Int)]], Int) = {
    n match {
      case Leaf => (Leaf, x)
      case Node(c, v, left, right) =>
        val (leftTree, myX) = calcPos(left, x, y+d)
        val (rightTree, nextX) = calcPos(right, myX+d, y+d)
        (Node(c, AugmentedData(v, (myX, y)), leftTree, rightTree), nextX)
    }
  }
  calcPos(node, start, start)._1
}
```

### Step 2 - Drawing the Tree

Once we have the positions, drawing a tree is trivial. A natural way to represent the tree is using [SVG](http://en.wikipedia.org/wiki/Scalable_Vector_Graphics) format. Since it is xml the tree structure can be directly represented. Scala also support XML literals which makes things easier.

```scala
def toSvg[T : Orderable](node: RBT[AugmentedData[T, (Int, Int)]],
                         width : Int = 640, height : Int = 480): NodeSeq = {

  val rgb = Map[Color, String](Black -> "#000000", Red -> "#7f0000")

  def genNode(n: RBT[AugmentedData[T, (Int, Int)]], x: Int, y: Int): NodeSeq =
    n match {
      case Node(_, AugmentedData(_, (lx, ly)), _, _) =>
        <line id="svg_1" y2={y.toString} x2={x.toString} y1={ly.toString}
              x1={lx.toString} stroke-width="3" stroke="#000000" fill="none"/> ++
        genSvg(n)
      case Leaf => <!-- leaf -->
  }

  def genSvg(n: RBT[AugmentedData[T, (Int, Int)]]) : NodeSeq = n match {
    case Node(c, AugmentedData(t, (x, y)), left, right) =>
      genNode(left, x, y) ++
      genNode(right, x, y) ++
      <ellipse stroke={rgb(c)} fill={rgb(c)} cy={y.toString} cx={x.toString}
               ry="20" rx="20" stroke-width="5"/> ++
      <text xml:space="preserve" text-anchor="middle" font-family="serif"
            font-size="24" y={(y+7).toString} x={x.toString} stroke="#ffffff"
            fill="#ffffff">{t}</text>
    case Leaf => <!-- leaf -->
  }
  <svg width={width.toString} height={height.toString} xmlns="http://www.w3.org/2000/svg">
    <g>
      <title>RBT</title>
      {genSvg(node)}
    </g>
  </svg>
}
```

This produces svg images like the following:

<img src="/img/r.svg" alt="1" style="width: 50%; height: 50%"/>

See the complete code [here](https://github.com/amitdev/functional-ds/blob/master/src/main/scala/ds/SetUtil.scala).
