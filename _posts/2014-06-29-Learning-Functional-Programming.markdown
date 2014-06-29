---
layout: post
title: Learning Functional Programming - A Roadmap
category: Coding
tags: functional-programming
year: 2014
month: 06
day: 29
published: true
summary: A learning roadmap for functional programming (FP). If you are new to FP or have intermediate knowledge, you might find this useful.
---

Introduction
------------

This is a learning roadmap for Functional Programming (FP). There are quite a lot of resources to learn FP on the web (just like on any subject). So the challenge is not to find a resource to learn, but to find *quality* resources. This post is an attempt to compile those (obviously based on what helped me, and might be subjective to some extent).

Starting Point
--------------

What you pick first is often important since unlearning is more difficult than learning. So start with the best - SICP. The [course videos](http://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-001-structure-and-interpretation-of-computer-programs-spring-2005/video-lectures/) as well as the [book](http://mitpress.mit.edu/sicp/full-text/book/book.html) are freely available online.


Other excellent courses
-----------------------

This can be followed up with some other good courses available online (at the time of writing):

- [The Paradigms of computer programming](https://courses.edx.org/courses/LouvainX/Louv1.01x/1T2014/info) has a section on FP. The course is also good overall, if you want to learn different paradigms of programming. Like SICP, the accompanying [text](http://mitpress.mit.edu/books/concepts-techniques-and-models-computer-programming) is also a masterpiece.
- [The functional programming course](https://www.coursera.org/course/progfun) offers a modern perspective based on [Scala](http://scala-lang.org/). Again highly recommended along with the [book](http://www.artima.com/shop/programming_in_scala_2ed).


Short bytes
-----------

If you are looking for short insights into why FP is important or some key aspects of it, following videos are recommended:

- [Simple Made Easy](http://www.infoq.com/presentations/Simple-Made-Easy)
- [Introduction to Haskel](http://www.haskell.org/haskellwiki/Video_presentations#Introductions_to_Haskell) has couple of FP talks.
- [Reactive Programming](http://www.youtube.com/watch?v=4L3cYhfSUZs) talk explains why FP is one of the best choices for [reactive](http://www.reactivemanifesto.org/) programming.


Thinking functionally
---------------------

After you learn the basics, the most important skill needed is to think functionally. The best way to grok that is to get your hands dirty - do the exercises, apply it on the next programming task you do, etc.

Following are some good books which will deepen your knowledge on FP:

- [Introduction to Functional Programming](http://www.amazon.com/Introduction-Functional-Programming-Haskell-Edition/dp/0134843460).
- [Learn you a haskell](http://learnyouahaskell.com/).

Developing purely functional data structures is another art to master. Okasaki's [Purely Functional Data Structures](http://www.amazon.com/Purely-Functional-Structures-Chris-Okasaki/dp/0521663504) is still the best resource in this area. This [post](http://cstheory.stackexchange.com/questions/1539/whats-new-in-purely-functional-data-structures-since-okasaki) highlights some of the other functional data structures.

Interesting Papers
------------------

- [Why FP matters](http://www.cse.chalmers.se/~rjmh/Papers/whyfp.pdf)
- [Essence of Monads](http://homepages.inf.ed.ac.uk/wadler/papers/essence/essence.ps). I found this to be the best introduction to Monads.
- [Original paper on Lisp](http://www-formal.stanford.edu/jmc/recursive/recursive.html)
- [All papers](http://www.cs.nott.ac.uk/~gmh/bib.html) by Graham Hutton are excellent. Its hard to pick and choose from them.
- Somewhat advanced papers on [functional pearls](http://www.haskell.org/haskellwiki/Research_papers/Functional_pearls).


Summing up
----------

These are just some pointers that came to my mind. If I missed anything important please let me know.

I hope this helps you in your functional programming journey!
