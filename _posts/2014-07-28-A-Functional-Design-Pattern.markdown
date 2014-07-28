---
layout: post
title: A functional design pattern
category: Coding
tags: functional-programming, python
year: 2014
month: 07
day: 28
published: true
summary: A simple introduction to Monads - seeing them as a design pattern without going through the theory.
---

Introduction
------------

This is not an attempt to be yet another Monad tutorial. There are probably more Monad tutorials than one can read already out there - see this [timeline](http://www.haskell.org/haskellwiki/Monad_tutorials_timeline) or do a google search. This rather is a simple introduction to understand Monads based on the [Essence of FP](http://homepages.inf.ed.ac.uk/wadler/papers/essence/essence.ps) paper. Not the theory, just the design idea which is arguably simpler to understand.

A basic interpreter
-------------------

Let's consider implementing a basic interpreter - one which can add numbers (the paper talks about a bigger one, but this simple example will serve our purpose). And we are concerned only with evaluation, we assume the AST is already available to us. So, we should be able to do things like:

```python
Num(1) == 1
Add(Num(2), Num(3)) == 5
Add(Num(10), Var("v")) {"v" : 5} == 15
```

In the last example, a variable is defined which should be available via an environment. Let us consider an implementation in Python.

<script src="https://gist.github.com/amitdev/200bd0bc38a503ed5d73/6df6227018ae357cb10c0b98b8dbadb7ca53d1fc.js"></script>

Note: Don't focus too much on the implementation - I've chosen the commonly used way in Python, it doesn't matter if you use objects or functions here.

Coming back to the code above, there are three objects: ``Num`` which evaluates to its value, ``Var`` which evaluates to value defined in env and ``Add`` which evaluates its left and right operands and adds them. Pretty simple.

Error Handling
--------------

Currently we don't have any error handling. What if ``Num`` contains something other than numbers? a variable is not defined? One option is to return something like ``None``, but that means every caller should check that explicitly. We could raise exceptions, but they don't compose well. Besides, currently all expressions evaluate to same type. Consistency is nice. So what could we do?

> All problems in computer science can be solved by another level of indirection - Butler Lampson

Ok. Let us try to abstract the return type. Currently it is returning the value (number). Instead let us return an object that contains the value. Maybe we can extend this object later. Following is the first iteration - without changing any behavior, but only wrapping the result in the new object.

```python
class M(object):
  def __init__(self, val):
    self.val = val

  def __repr__(self):
    return "%s(%r)" % (self.__class__.__name__, self.val)

  def __eq__(self, other):
    return self.val == other.val

def unit(v):
  return M(v)

def bind(m, f):
  return f(m.val) 
```

So ``M`` is our container object. We also added two utility functions: ``unit`` which just creates ``M``, and ``bind`` which applies a given function to value in ``M``. Only changes to our interpreter is in ``evaluate`` methods:

```python
# In Num.evaluate:
  return unit(self.val) # Instead of just self.val

# In Var.evaluate:
  return unit(env[self.name]) # Instead of just env[self.name]

# In Add.evaluate:
  return bind(self.left.evaluate(env), lambda x: bind(self.right.evaluate(env), lambda y: unit(x+y)))
```

The change to ``Add`` might seem complicated, so lets go through it. In ``evaluate``, previously we get two numbers and add them. But now we get two ``M`` objects. Say, we got ``M(2)`` and ``M(3)``. How will we make an ``M(5)`` out of it? That is where the ``bind`` function comes in. It helps us to peek inside ``M`` and access the value. So in the example, the above code is equivalent to:

```python
# bind(M(2), lambda x: ..)
#	=> {x = 2} bind(M(3), lambda y: ...)
#	=> {x=2, y=3} unit(x+y) 
#	=> unit(2+3) 
#	=> M(5). 
```

I hope it is clear now, otherwise try to take it apart yourself. So our interpreter still has the same functionality, but returns a new type now:

```python
Num(1) == M(1)
Add(Num(2), Num(3)) == M(5)
Add(Num(10), Var("v")) {"v" : 5} == M(15)
```

Now that we have a generic return type, we could add error as a first class type.

```python
class Error(M):
  pass 
```

We can start handling errors like so:

```python
# In Num.evaluate:
  if isinstance(self.val, Number):
    return unit(self.val)
  else:
    return Error("%s is not a number" % self.val) 
```

We need to do one last thing. ``bind`` on an ``Error`` should pass it on instead of applying the operation:

```python
def bind(m, f):
  if isinstance(m, Error):
    return m
  return f(m.val) 
```

That's all. Now our interpreter can handle errors and still maintain the same interface. Let us call this design technique *Monads*. In theory there are [laws](http://en.wikipedia.org/wiki/Monad_%28functional_programming%29#Monad_laws) governing Monads, but let's ignore them for now.

> Monads are return types that guide you through the happy path - Erik Meijer

See the complete code [here](https://gist.github.com/amitdev/200bd0bc38a503ed5d73/d1cb471cf4809d3112363720b2baf50a748e842b). 

Adding State
------------

What if we need to add some state to our interpreter? Say we need to keep track of number of times add was performed (it could be any state that we want to maintain). One option is to add global variable to keep track, but that is evil. Since we already have abstracted the return type, let us see if we can utilize that. One way to model it is to return a pair of values - the result and count. For eg:

```python
Num(1) == State(1, 0)                           # 0 addition
Add(Num(2), Num(3)) == State(5, 1)              # 1 addition
Add(Num(2), Add(Num(3), Num(5))) == State(5, 2) # 2 addition
```

So we will define ``State`` which encapsulates the data we need to track. It will a lazy object that when invoked will return the actual state.

```python
class State(M):
  def __init__(self, val, count=0, old=None, f=None):
    super(State, self).__init__(val)
    self.count = count
    self.old = old
    self.f = f

  def __call__(self, count):
    if not self.old:
      return State(self.val, count)
    else:
      new_state = self.old(count)
      return self.f(new_state.val)(new_state.count)

  def __eq__(self, other):
    return self.val == other.val and self.count == other.count

class Counter(State):
  def __call__(self, count):
    return State(self.val, count+1)
```

The ``unit`` and ``bind`` now return ``State`` objects:

```python
def unit(v):
  return State(v)

def bind(m, f):
  if isinstance(m, Error):
    return m
  return State(m.val, old=m, f=f)
```

The only change required in the interpreter is in ``Add``:

```python
# In Add.evaluate
  return bind(self.left.evaluate(env),
	      lambda x: bind(self.right.evaluate(env),
                             lambda y: bind(Counter(None), 
                                            lambda t: unit(x+y))))
```

An additional bind call is added to get the result (``unit(x+y)``) and apply ``Counter`` to maintain count. The complete code is [here](https://gist.github.com/amitdev/200bd0bc38a503ed5d73).

Conclusion
----------

You probably are aware of higher order functions like ``map`` which takes a function that acts on values inside a *container* and produce new values. But the shape of the *container* itself does not change. With Monads, the function (in ``bind``) acts on value inside the *container* but returns a *new container*. This is where Monads get the power from. Languages like Haskell, Scala etc provide rich set of generic operations and syntactic sugar around it.

In any case, considering it purely as a design approach, this is simple and something anyone could've invented.
