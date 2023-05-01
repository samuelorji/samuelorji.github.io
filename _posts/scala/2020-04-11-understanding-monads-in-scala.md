---
layout: post
title:  "Understanding Monads in Scala"
date:   2020-04-11 12:37:00 +0100
tags: monads scala functional-programming
---

This post is going to try to simplify the concepts of monads, this means that a lot of technicalities may be omitted, but I'll be sure to attach links to more technical concepts in this post.

Now, what on earth is a monad?. A more formal definition according to [this](https://wiki.haskell.org/All_About_Monads#What_is_a_monad.3F) is

`A monad is a way to structure computations in terms of values and sequences of computations using those values. Monads allow the programmer to build up computations using sequential building blocks, which can themselves be sequences of computations. The monad determines how combined computations form a new computation and frees the programmer from having to code the combination manually each time it is required.`

A slightly informal definition is that a Monad is a **Wrapper** that wraps objects in such a way that you can then easily compose or combine computations of that wrapper type.

according to Mr John Degoes,

> Monads are a "functional design pattern" that come up a lot when you're writing purely-functional code. There's a good reason for their ubiquity: monads encapsulate the essence of sequential computation.

In Scala, these monads can be used in a for comprehension to effectively compose functions. Here's a little controversial saying,

_**'The only thing monads are relevant for, from a Scala language perspective, is the ability of being used in a for-comprehension'**_

Now, how do we create a Wrapper, or a Monad. For now, let's say that any class that implements the following methods `map` and `flatMap` can be regarded as a Monad. Although this isn't entirely true, as there are some laws that must be obeyed, as well as some other technicalities that are involved, but for the sake of this tutorial, we are satisfied with just a `map` and `flatMap` implementation.

To get a practical understanding of Monads, and effectively use them in a for comprehension, Let us imagine this scenario. We go to the supermarket to buy `Nuts`, and each type is wrapped up in a Paper `Bag`, Now, let's say we pick a different type of nuts each in their own separate bags and we want to calculate the price of all the different bags of nuts we have, or we throw one bag of nut into another bag of nut, or at a certain point, we decide that we want to reduce the number of nuts in a bag. One way to go about it is to manually unwrap the nuts, perform the maybe complex operations and then wrap them up again in a paper bag because, well you can't carry that many nuts in your hands can you?

![](/assets/images/joey.jpg)

Monads, help us abstract over the wrapping and unwrapping process and just makes you think only about the values contained in the bag in a for comprehension. Enough talk, Let's see some code.

To model our Monad, The blueprint of any monad is shown below, where Monad represents the class we want to make a Monad

{% highlight scala %}
 Monad[A] {
  def map[B](f : A => B) : Monad[B]
  def flatMap[B](f : A => Monad[B]) : Monad[B]
}
{% endhighlight %}

> The `map` method is basically you exposing an API for the user of this monad to manipulate the content of the Monad, which in our case will be a `Nut`, Note that all you can do is modify the contents of the Monad, and thereafter the result will be wrapped for you. The `flatMap` method, on the other hand, which is more powerful exposes an API for you to manipulate the contents of the current monad for you to use to generate a new Monad, it's a method to produce the _next_ computation from the value of the _current_ computation , so in our case, we can take the contents of a bag and not just modify the contents (like taking a part of the nuts we want, taking from some other types of nuts), but we then return any other kind of bag if we want containing any kind of nut.

And type A represents the value we want to wrap, in our case, Nuts is modeled thus.

{% highlight scala %}
sealed trait Nuts{
  val quantity : Int
  private val price : Double = 2.0
  def total  =  quantity * price
   }
case class Peanut(quantity : Int) extends Nuts
case class GroundNut(quantity : Int) extends Nuts
case class CashewNut(quantity : Int) extends Nuts
case class MixedNuts(quantity : Int) extends Nuts
{% endhighlight %}

Now, let's create our Bag that should wrap these nuts and should also be a Monad

{% highlight scala %}
case class Bag[A](item: A) {
  def map[B](f: A => B): Bag[B] = Bag(f(item))
  def flatMap[B](f: A => Bag[B]): Bag[B] = f(item)
}
object Bag {
  def unit[A](v: A) = Bag(v) 
}
{% endhighlight %}

With the `map` and `flatMap` method, we can easily just operate on the contents of the bag of nuts without having to unwrap the bag, and then wrap it again. If you closely observe the `map` and `flatMap` method, they take care of the wrapping and unwrapping for us.

> There's a companion object with a `unit` method, which is not necessarily needed for a Monad in our context, but if you are interested, in the latter part of this post, I'll use it to prove the laws that our Monad class actually satisfies

Now, let's go buy 3 types of nuts, and for some weird reason, when we are at checkout, we apply a discount to the first, remove 20 pieces from the second and double the quantity of the third. Modelling this with some utility functions, we have

{% highlight scala %}
val cashewBag = Bag(CashewNut(40))
val groundNutBag = Bag(GroundNut(40))
val peanutBag = Bag(Peanut(40))

/** you pay for only 3/4 of the number of nuts 
 *  if you have more than 20 nuts
 */
def applyDiscount(quantity : Int) : Int =
 if(quantity > 20)(0.75 * quantity ).toInt  else quantity  

def printTotal(nut : Nuts) : Unit =
 println(s"total is  ${result.item.total}")
 {% endhighlight %}

Now, If our Bag type wasn't a monad, the cleanest, regular non-monadic way to do this would have been this

{% highlight scala %}
val newCashewBag = Bag(CashewNut(applyDiscount(cashewBag.item.quantity)))

val newGroundBag = Bag(GroundNut(groundNutBag.item.quantity - 20))
val newPeanutBag = Bag(Peanut(peanutBag.item.quantity * 2))
val mixedNuts    = Bag(MixedNuts(newCashewBag.item.quantity +
                                   newGroundBag.item.quantity +
                                   newPeanutBag.item.quantity))

printTotal(mixedNuts.item)
{% endhighlight %}

Looking at this code, we can see that it's not easily composable, there is always a call to get the item which unwraps the bag, performing some computation and then wrapping the result in a new Bag as underlined. We can also see that we are creating unnecessary `val` variables for each wrapping and unwrapping. Now, imagine we had over 100 kinds of nuts, this would easily have been a nightmare.

But since our Bag class is a Monad, and because we have declared methods for manipulating contents of our monad, we see that we can easily and sequentially compose our computations without having to deal with wrapping and unwrapping anything or declaring unnecessary `val` variables.

{% highlight scala %}
val result  =  for {
    a <- cashewBag.map(x => CashewNut(applyDiscount(x.quantity)))
    b <- groundNutBag.map(x => GroundNut(x.quantity + 20))
    c <- peanutBag.map(x => Peanut(x.quantity * 2))
  } yield { MixedNuts(a.quantity + b.quantity + c.quantity) }

  printTotal(result.item)
{% endhighlight %}

The beauty of this is that it's so clean and composable, the fact that the Monad has abstracted over wrapping and unwrapping its contents and just exposed the methods `map` and `flatMap` to deal with the contents of the wrapper.

**Conclusion**

And that's it, Monads are really this easy, this example may be a little trivial, but let's imagine if the scala `Future` class wasn't a Monad, take a moment to imagine you couldn't call `map` and `flatMap` on a `Future`, you would easily see that it would have been a nightmare, having to deal with callback hell. Also, imagine there was no `map` and `flatMap` methods on lists, or Options.

Behind the scenes, in a for comprehension, the compiler actually uses the `map` and `flatMap` method, to see that, try compiling your class with `scalac -Xprint:parse`

**Extras**

If you remember that I said for a class to be a monad, [there are some laws that must be obeyed](https://en.wikipedia.org/wiki/Monad_(functional_programming)#Monad_laws). If you are interested in this, you can continue to read further. But here are the laws really simplified using Scala.

**MONAD LAWS**

Before, we explain these laws, remember we defined a method called `unit` which some other people call `identity` or `point` e.t.c. It serves just a basic purpose as described below.

> **The `unit` method is basically just used to wrap an object into a wrapper of that type, in our case, we can supply a `Nut` to the unit method, and it wraps it in a `Bag` for us as a `Bag[Nut]`**

The three monad laws according to the Haskell Wiki are informally expressed as

1. Left Identity
2. Right Identity
3. Associativity

As we test for these laws, let's define some functions `f`and `g` which are of type `A -> Monad[A]`, these are just functions that take a type A and return a Monad of type A, we will also define an object of type `Monad[A]` to carry out these tests on. In our case, `A` is `Nuts`.

{% highlight scala %}
val cashew  = CashewNut(10)      // value
val cashewBag = Bag(cashew)     // test Monad instance

val f: CashewNut => Bag[CashewNut] = nut => Bag(CashewNut(nut.quantity + 30))

val g: CashewNut => Bag[CashewNut] = nut => Bag(CashewNut(nut.quantity \* 50))
{% endhighlight %}

I am testing on `CashewNut`, but the law will apply to all types of nuts. Now let's see what these laws are

**LEFT IDENTITY**

So, if we have some basic value _x_, a monad instance _m_ (holding some value) and functions _f_ and _g_ of type _Int → M\[Int\]_, we can write the laws as follows:

The left identity law states that a value `x`, and our defined
functions `f` and `g`, the following must hold 
_**unit(x).flatMap(f) == f(x)**_

Testing this law we have

println(Bag.unit(cashew).flatMap(f) == f(cashew)) // prints true

**RIGHT IDENTITY**

The right identity law states if we have a monad instance `X`**,**
calling `flatMap` on that instance and passing a `unit` should
return the same monad instance `X`, mathematically represented as
**X.flatMap(unit) == X.** 

Let's test this also
{% highlight scala %}
println(cashewBag.flatMap(Bag.unit) == cashewBag)  // prints true 
{% endhighlight %}

**ASSOCIATIVITY**

This law states that for a monad instance `m` and with our already
defined functions `f` and `g`, the following must hold
_**m.flatMap(f).flatMap(g) == m.flatMap(x ⇒ f(x).flatMap(g))**_

Testing this law,

{% highlight scala %}
println(cashewBag.flatMap(f).flatMap(g) == cashewBag.flatMap(x ⇒  f(x).flatMap(g)))  // prints true
{% endhighlight %} 

We can clearly see that the Monad we just created is in fact, a mathematically correct Monad.
