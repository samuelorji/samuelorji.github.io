---
layout: post
title:  "Essential Effects 06: Integrating with Legacy systems"
date:   2021-04-11 12:37:00 +0100
tags: cats fp functional-programming scala essential-effects
---
### Integrating Asynchrony

We've dealt a lot with IO values, but in practice, we may have to deal with legacy  
codebases that use futures or other async structures. But since we are writing pure functional programs, we need a way to be able to capture or lift the result of these asynchronous computations into an IO.

Cats effect provides that capability via the `IO.async` method. The way this works is that the method takes a function from a callback to Unit, where the callback itself is a function from an Either[Throwable,A] to Unit.

The function signature looks like this

`def async[A](k : (Either[Throwable,A] => Unit ) => Unit)`

The function signature may seem a little daunting, but the basic idea is that you call the method and complete the callback at the end of the computation with either a `Left` or a `Right`

Now, for a contrived example, let's imagine we have an asynchronous method that returns an Int, and we want to lift that into an IO, we could do it like this:
{% highlight scala %}
object IOAsyncExample extends IOApp {
  val ec = Executors.newCachedThreadPool()

  def getMagicNumber() : Int = {
    Thread.sleep(500)
    43
  }

  def asynComputation =  ec.submit{
    new Callable[Int] {
      override def call(): Int = getMagicNumber()
    }
  }

  val magicIO =  IO.async[Int]{ cb =>
    try cb {
      val result = Right(
        asynComputation.get(1, TimeUnit.SECONDS)
      )
      ec.shutdown()
      result
    }
    catch {
      case NonFatal(e) => cb(Left(e))
    }
  }

  override def run(args: List[String]): IO[ExitCode] = {
    for {
      _           <- IO(println("Let's calculate the magic number"))
      magicNumber <- magicIO
      _           <- IO(println(s"magic number is $magicNumber"))
    } yield ExitCode.Success
  }
}
{% endhighlight %}

With this, we've been able to encapsulate or lift our async computation into an IO. This is more or less the basis for the utility method to lift a future into an IO: `IO.fromFuture`

> IO.async provides a callback cb so the API (however asynchronous) can signal the result of the computation. When the API computes the result it provides it to the callback
