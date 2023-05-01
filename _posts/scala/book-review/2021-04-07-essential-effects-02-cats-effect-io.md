---
layout: post
title:  "Essential Effects 02: Cats Effect IO"
date:   2021-04-07 12:37:00 +0100
tags: cats fp functional-programming scala essential-effects
---

### Constructing an IO

An IO can be constructed using IO.delay to capture side effects as an IO value

`val stringIO : IO[String] = IO.delay("Hello World!")`

Most often, people will use the apply method which internally just calls the delay method.

`val stringIO : IO[String] = IO("Hello World!")`

Similarly, we could also represent exceptions as IO values by “lift”ing an exception into IO,  
as long as we provide the “expected” type of the IO had it succeeded,  
either explicitly or through type inference:

`val res: IO[String] = IO.raiseError(new Exception("oops !"))`

We could also transform futures into IO, which makes it really easy to integrate with legacy codebase:

{% highlight scala %}
def futureLong : Future[Long] = ???
val ioFromFuture = IO.fromFuture(IO(futureLong))
{% endhighlight %}

It may ask for an implicit context shift, which is provided if run with the cats IOApp instead of the scala native App

### Transforming IO:

The IO type is a functor, so we can map over it:

 `IO(3).map(_ + 1) // IO(4)`

It is also an applicative functor, so we can combine and operate on multiple values:

 `(`IO(3),IO(4),IO(5)).mapN(_ + _ + _) // IO(12)`

It is also a Monad, so we can flatMap over it as well as use it in a for comprehension:

{% highlight scala %}
for {
  i <- IO(3)
  j <- IO(4)
} yield i +  j // IO(7)
{% endhighlight %}

### Error Handling:

Similar to Futures, we can raise and deal with errors in IO:

`val result IO.raiseError[Int](new Exception("oops !")) // similar to Future.failed`

we can also recover from errors too:

{% highlight scala %}
result.handleError(_ => 40) // similar to Future#recover 
result.handleErrorWith(_ => IO(40)) // simialr to Future#recoverWith 
{% endhighlight %}

If you want to your error type to be explicitly seen in the type signature, we cal easily call IO#attempt  
which returns an IO[Either[Throwable,A]]:

`val attemptedResult : IO[Either[Throwable, Int]] = result.attempt `

Error-handling Decision Tree

- If an error occurs in your IO[A] do you want to…  
    perform an effect? use:

   `onError(pf: PartialFunction[Throwable, IO[Unit]]): IO[A]`

- transform any error into another error? use

   `adaptError(pf: PartialFunction[Throwable, Throwable]): IO[A]`

- transform any error into a successful value? use:

   `handleError(f: Throwable ⇒ A): IO[A]`

- transform some kinds of errors into a successful value? use:

   `recover(pf: PartialFunction[Throwable, A]): IO[A]`

- transform some kinds of errors into another effect? use:

   `recoverWith(pf: PartialFunction[Throwable, IO[A]]): IO[A]`

- make errors visible but delay error-handling? use:

  `attempt: IO[Either[Throwable, A]]`

Otherwise, use

`handleErrorWith(f: Throwable ⇒ IO[A]): IO[A]`

### Executing IO Values

We've seen that IO values delay or suspend computation, and if we want to run those computations,  
we can easily call `unsafeRunSync` or `unsafeRunAsync` or `unsafeToFuture` which is used when we want to convert  
from `IO` to `Future` when integrating with legacy codebase.

But it's advisable to use the IOApp provided by cats for running Effects.
