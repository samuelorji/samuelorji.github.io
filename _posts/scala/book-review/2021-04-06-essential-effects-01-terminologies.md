---
layout: post
title:  "Essential Effects 01: Terminologies"
date:   2021-04-06 12:37:00 +0100
tags: cats fp functional-programming scala essential-effects
---
### Higher Kinded type

A higher-kinded type which is also called a type constructor describes types that require a type to be a concrete type.

An example is the Option data type:

```scala
val something:  Option =  ??? // not a valid type
val otherthing : Option[Int] = ??? // valid type 
```

Same goes for list, futures e.t.c

### Functors

A functor captures the notion of something you can map over, changing its  
“contents” (or output) but not the structure itself.  
The type signature of map on higher kinded types F[A] is:

def map[B](f : A => B) : F[B]

### Applicatives

An applicative functor, also known as applicative, is a functor that can transform  
multiple structures, not just one.  
Imagine, we have a number of Option[Int] and we want to perform some sort of operation all of them in a functional manner, we could do that  
using a for loop:

```scala
val opt1 = Option(1)
val opt2 = Option(2)
val opt3 = Option(3)

val result = for {
  a <- opt1 
  b <- opt2 
  c <- opt3 
} yield a + b + c
```

But with an applicative functor, we can combine the operations using one of the `mapN` methods.

`val result = (opt1,opt2,opt3).mapN((a,b,c) => a + b + c)`

or simply

`val result = (opt1,opt2,opt3).mapN(_ + _ +_ )`

We've done the same thing with less lines of code. The applicative functor has enabled us to combine multiple F[A]

### Monads

A monad is a mechanism for sequencing computations: this computation happens after that computation. Roughly speaking, a monad provides a flatMap method for a context F[A]:

`def flatMap[B](f: A => F[B]): F[B]`

The for comprehension as well know is just syntactic sugar for nested flatMap calls

Effects can be composed using the `flatMap` method.
