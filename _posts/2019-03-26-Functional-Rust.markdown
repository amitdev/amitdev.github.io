---
layout: post
title: Functional programming in Rust
category: Coding
tags: rust functional-programming
year: 2019
month: 03
day: 26
published: true
summary: Introduction to functional style programming in Rust
---

## Rust Overview

[Rust](https://www.rust-lang.org/) is a modern, multi paradigm language with the following primary goals:
* Performance (zero cost abstractions)
* Reliability (no data races)

Rust is strongly influenced by functional programming languages like ML, so it is
possible to follow a functional coding style in it. 
However being functional is not one of Rust's explicit goals.

In this post (and possibly follow up ones) I'll explore functional programming in  Rust. This is not an introduction to
Rust or functional programming. Basic familiarity with them is assumed.

## Functional Programming
The goal of programming is to manage complexity.

In functional programming we manage complexity by principled composition - large things are composed of
small things.

The more things we can compose together the better abstractions
we can build and reduce complexity. The building blocks are types and functions, and let us
see how well we can compose them in Rust.

## Algebraic data types

Lets start with types. One way to classify types is by looking at number of inhabitants,
that is, how many values of a particular type exists. For eg, the unit type (`()` in Rust) has only
one value, `bool` has two values etc. Apart from builtin types, Rust supports Sum and Product types, which means we can
combine basic types using addition or multiplication (in terms of number of inhabitants). The key
insight here is that we can use the familiar algebra from maths to reason about them, hence the name.

In Rust we could create a sum type
by using `enum`. Think of it more like type constructor in Haskell, than enum in C/Java.

For eg. we can create a type with a two values as follows:

{% highlight rust %}
enum Wing { // name of type
    Left    // value
    Right   // value
}
{% endhighlight %}

A product type can be created using a tuple `(A, B)` or `struct`.

Here are examples of some types:


| # of Inhabitants | Rust | Haskell | 
| ---- | ----------- |
| 0 | `enum Void {}` |  `data Void` or <br> `Data.Void`|
| 1 | `enum Unit { Unit }` or <br> `()` |  `data Unit = Unit` or <br> `()` |
| 1 + 1 | `enum Direction { Up, Down }` or <br> `bool` | `data Direction = Up | Down` or <br> `Boolean` |
| 1 + A | `enum Option<A> { Some(A), None }` | `data Maybe a = Just a | Nothing` |
| A + B | `enum Result<A, B> { Ok(A), Err(B) }`| `data Either a b = Left a | Right b` |
| A * B | `(A, B)` or <br> `struct Tuple { a: A, b: B }` | `data (a, b) = (a, b)` |

As you can see we can compose types using addition or multiplication. This is useful in modelling
types.

For eg. since `A + A = 2 * A`, we could represent a type `enum Score<A> { Left(A), Right(A) }`
as `(bool, A)`. Intuitively, the boolean represents whether the value is left or right.

## Pattern matching
A language feature that goes well with algebraic data types in pattern matching. It helps to deconstruct
data out of types in a simple way. Rust offers powerful pattern matching support and almost anything can
be deconsutructed using it. For example with the following types:

{% highlight rust %}
enum Maybe<A> {
    Just(A),
    Nothing
}

struct Person {
    name: String,
    age: u8,
    email: Maybe<String>
}
{% endhighlight %}

we can pattern match things out of it using:

{% highlight rust %}
let person = Person {
  name: "Alice".to_string(),
  age: 25, 
  email: Just("alice@example.com".to_string())
};
match person {
    Person { name : name, age: age, email : Just(email) }
      => println!("Person {}, aged {} with email {}", name, age, email),
    Person { name: name, age: age, email : Nothing }
      => println!("Person {}, aged {} with no email", name, age)
}
{% endhighlight %}

Pattern guards and capturing pattern in a single variable is also supported. For eg:

{% highlight rust %}
match someVar {
    Some(val @ 0 ... 9) => println!("within 0 to 9, {}", val),
    Some(val) if val > 20 => println!("greater than 20, {}", val),
    Some(val) => println!("Should be less than 20, {}", val),
    None => println!("None")
}
{% endhighlight %}

Patterns is commonly used in match expressions but they can be used in place of identifier in other places as well like
variable declaration, function arguments etc.

## Functions

Functions are first class in Rust and it supports [closures](https://doc.rust-lang.org/book/ch13-01-closures.html) as well.

However, one has to think about ownership and lifetime of variables when dealing with them.
Basically Rust is not a language with garbage collection and the programmer has to
give hints to the compiler about ownership of variables so they can be *dropped* after use. This is
probably the hardest part to get while learning Rust and also what makes Rust special.

Read more about ownership [here](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html).

Lets start with a simple definition of factorial:

{% highlight rust %}
fn factorial(n: u64) -> u64 {
    match n {
        0 | 1 => 1,
        n     => n * factorial(n - 1),
    }
}
{% endhighlight %}

This is a straightforward translation of recursive definition of factorial; `factorial(n) = n * factorial(n-1)`.
While this is fairly clear, it is often better to use high level combinators than explicit recursion.
So we can write above as:

{% highlight rust %}
fn factorial(n: u64) -> u64 {
    (1..n+1).fold(1, |acc, i| acc * i)
}
{% endhighlight %}

where `|acc, i| acc * i` is a closure (similar to `\acc i -> acc * i` in Haskell, or `(acc, i) -> acc * i` in Java).
So the above code basically takes range of numbers from 1 to n inclusive and multiplies them.

Note: Most of the operators in Rust have traits associated with it. So the `*` operator above is defined in
`std::ops::Mul` trait and `acc * i` is a short hand for `acc.mul(i)`. So the above can also be written as:

`(1..n+1).fold(1, Mul::mul)`

This makes it clear that `factorial(n)` is the product of first n natural numbers. It can be made
explicit by using the builtin `product` function, which folds over using multiplication.

{% highlight rust %}
fn factorial(n: u64) -> u64 {
    (1..n+1).product()
}
{% endhighlight %}

The key here is all of the above does not have any performance overhead, it performs similar to an
imperative loop that multiplies the numbers.

## Higher order functions

Now lets look at how to define and use higher order functions. Function types in Rust implement
the `Fn` trait (think of traits as something similar to type classes). A function that takes in
type `X` and return type `Y` can be represented as `Fn(X) -> Y`.

For example, consider a function that takes a function and calls it with given argument:

{% highlight rust %}
fn apply<F, A, B>(f: F, arg: A) -> B 
where F: Fn(A) -> B {
    f(arg)
}

fn square(a: i32) -> i32 { a * a }

// apply can take another function as argument
let n: i32 = apply(square, 4);    // n = 16
// apply can also take a closure as argument
let n: i32 = apply(|x| x * x, 4); // n = 16
{% endhighlight %}

A more interesting example of higher order function is the fixed point function. A fixed point of a function `f` is a value `a` such that `f(a) == a`.
One way to define the fixed point function is provided in [sicp](https://mitpress.mit.edu/sites/default/files/sicp/full-text/book/book-Z-H-12.html#%_sec_1.3.3).
We can translate that to Rust almost verbatim by adding in the right types.

{% highlight rust %}
const TOLERANCE: f64 = 0.000001;

fn fixed_point<F>(f: F, first_guess: f64) -> f64
    where F: Fn(f64) -> f64 {

    // Check if v1 and v2 are close enough to be considered equal
    fn close_enough(v1: f64, v2: f64) -> bool {
        (v2 - v1).abs() < TOLERANCE
    }

    // Call f(guess) and see if it is close enough to guess. If so
    // we have found the fix point of f
    fn try_guess<F>(guess: f64, f: F) -> f64 
        where F: Fn(f64) -> f64 {
        let next = f(guess);
        if close_enough(guess, next) {
            guess
        } else {
            try_guess(next, f)
        }
    }

    try_guess(first_guess, f)
}

fn sqrt(x: f64) -> f64 {
    fixed_point(|y| (y + x/y) / 2.0, 1.0)
}
{% endhighlight %}

How about functions returning functions? One way is to return a closure, but it requires little more work
that you would expect. For example, consider the `compose` function that composes two functions `f` and `g`:

{% highlight rust %}
fn compose<X, Y, Z, F, G>(f: F, g: G) -> impl Fn(A) -> C
where F: Fn(X) -> Y, G: Fn(Y) -> Z {
    move |x| g(f(x))
}
{% endhighlight %}

This might look unfamiliar, so lets go through it:

* `X, Y, Z, F and G` are type parameters. `F` is type of a function that takes an argument of type `X` and returns `Y` etc.
* `impl Fn(X) -> Z`: The function returns a closure. Since the actual type that is returned might vary per call
   (each closure is a differnt type since the capturing variables might differ), we need to tell compiler that what we really return is an implementation of the `Fn` trait.
* `move` This is related to ownership. Since we return a closure, we don't know when that is called and its lifetime may outlive the function. So the closure needs to own
the variables instead of borrowing it. This might seem confusing at first, but rust compiler has excellent [error messages](https://doc.rust-lang.org/error-index.html#E0373) to guide you here.

Note: There is also [FnOnce](https://doc.rust-lang.org/std/ops/trait.FnOnce.html) which only can be called once, and [FnMut](https://doc.rust-lang.org/std/ops/trait.FnMut.html)
which allows the closure to mutate state.

## Conclusion
Hope you got a basic taste of functional programming in Rust. Rust has pretty good support for functional programming
compared to languages like Java or C++, but not as much as a pure functional language like Haskell.
Rust is a good choice if performance and reliability
are what you are after with good support for building composable software. It has excellent [documentation](https://doc.rust-lang.org/), good community and a
friendly compiler.
