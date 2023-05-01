---
layout: post
title:  "Essential Effects 08: Concurrent Coordination"
date:   2021-04-13 12:37:00 +0100
tags: cats fp functional-programming scala essential-effects
---


It's bad to use vars to define state that's going to be shared by multiple threads, the same  
goes for using vars for using vars for sharing state amongst multiple effects.
{% highlight scala %}
object Refs extends IOApp {

  var counter = 0
  def tickingClocks(name : String) : IO[Unit] = {
    for {
      _ <- IO(println(s"[$name] : ${System.currentTimeMillis()}"))
      _ =  counter = counter + 1
      _ <- IO.sleep(1 second)
      _ <- tickingClocks(name)
    } yield ()
  }

  def printCounter() : IO[Unit] =
    for {
      _ <- IO(println(s"counter is $counter"))
      _ <- IO.sleep(1 second)
      _ <- printCounter()
    } yield ()

  val program : IO[ExitCode] = (tickingClocks("first clock"), tickingClocks("second clock"), printCounter()).parTupled.as(ExitCode.Success)

  override def run(args: List[String]): IO[ExitCode] = program
}
{% endhighlight %}

Running the above code will result in lost updates as the counter will never be accurate.

Although there are Atomic classes to solve these problems, they are not functional structures. Cats effect provides a functional wrapper around Atomic Classes called `Ref` .It has API's similar to Atomic Classes.

We can modify our example to use a `Ref` and guarantee that state is safely shared amongst effects and ultimately threads.
{% highlight scala %}
 def tickingClocks(name : String, counter : Ref[IO,Long]) : IO[Unit] = {
      for {
        _ <- IO(println(s"[$name] : ${System.currentTimeMillis()}"))
        _ <- counter.update(_ + 1)
        _ <- IO.sleep(1 second)
        _ <- tickingClocks(name,counter)
      } yield ()
    }

    def printCounter(counter : Ref[IO,Long]) : IO[Unit] =
      for {
        counterValue <- counter.get
        _            <- IO(println(s"counter is $counterValue"))
        _            <- IO.sleep(1 second)
        _            <- printCounter(counter)
      } yield ()

  override def run(args: List[String]): IO[ExitCode] =
    for {
      ref <- Ref[IO].of(0L)
      _   <- (tickingClocks("first clock",ref), tickingClocks("second clock",ref), printCounter(ref)).parTupled.as(ExitCode.Success)
    } yield ExitCode.Success
{% endhighlight %}

#### Deferred

Now, let's imagine we want to alert the user when the counter hits 13. One way to do this is to check for the value each time we update the counter. A better way to do this is by the use of the `Deferred` data type.

`Deffered` is a functional concurrency construct that sleeps or halts execution of subsequent effect until the task or value that was deferred completes. We can think of it as a Promise in Scala, that is completed elsewhere, could be another effect or effectively, another thread.

> Deferred gives us the ability to serialize the execution of an effect with respect to some newly-produced state

By using `Deferred`, we can separate the task to run upon completion from the task that completes the `Deferred`.

In our previous example, we'll use the `Deferred` data type to alert us when the counter reaches 13, that will be separate from other tasks.
{% highlight scala %}
    def tickingClocks(name : String, counter : Ref[IO,Long], alerter : Deferred[IO,Unit]) : IO[Unit] = {
      for {
        _            <- IO.sleep(1 second)
        _            <- IO(println(s"[$name] : ${System.currentTimeMillis()}"))
        counterValue <- counter.updateAndGet(_ + 1)
        _            <- if (counterValue >= 13) alerter.complete(()).attempt.void else IO.unit
        _            <- tickingClocks(name,counter,alerter)
      } yield ()
    }

  def alertIf13(is13 : Deferred[IO,Unit]) : IO[Unit] = {
    for {
      _ <- is13.get
      _ <- IO(println("ALERT!!!!!!!"))
    } yield ()
  }

    def printCounter(counter : Ref[IO,Long]) : IO[Unit] =
      for {
        counterValue <- counter.get
        _            <- IO(println(s"counter is $counterValue"))
        _            <- IO.sleep(1 second)
        _            <- printCounter(counter)
      } yield ()

  override def run(args: List[String]): IO[ExitCode] =
    for {
      ref     <- Ref[IO].of(0L)
      alerter <- Deferred[IO,Unit]
          _   <- (
            tickingClocks("first clock",ref, alerter), 
            tickingClocks("second clock",ref, alerter),
            alertIf13(alerter), 
            printCounter(ref)
            ).parTupled
    } yield ExitCode.Success
{% endhighlight %}

In this example, we complete the `Deferred` with `Unit` in the `tickingClocks` while we use it in the `alertIf13` function.

> We used an IO#attempt on the call to complete the Deferred because our `tickingClocks` function is recursive and will attempt to call complete on an already completed `Deferred` [which will lead to an IllegalStateException](https://github.com/typelevel/cats-effect/blob/1846813109b1e78c5bf36e6e179d7a91419e01d0/core/shared/src/main/scala/cats/effect/concurrent/Deferred.scala#L68)

#### Concurrent State Machines

An example used by the book was to design a functional [Countdown latch](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/CountDownLatch.html) using these functional concurrency structures.

To adapt to the parallel ticking clock example, here's how the latch could work:
{% highlight scala %}
def run(args: List[String]): IO[ExitCode] =
    for {
      latch <- CountdownLatch(13)
      _     <- (
            tickingClocks("first clock", latch),
            tickingClocks("second clock", latch),
            alertIf13(latch),
            ).parTupled
    } yield ExitCode.Success
{% endhighlight %}

Now, this was my initial implementation which didn't work because for some reason, calling complete on the `Deferred` didn't work:
{% highlight scala %}
object CountdownLatch {
    def apply(n : Int) : IO[CountdownLatch] = {
      require(n > 0 , "number of latches should be greater than 0")
      for {
        latchSignal <- Deferred[IO,Unit]
        latchState   <- Ref[IO].of[LatchState](CountingDown(n))
      } yield new CountdownLatch {
        override def await(): IO[Unit] =
          latchSignal.get

        override def decrement(): IO[Unit] = {
          latchState.update {
            case CountingDown(1) =>
              // last latch
              latchSignal.complete(())
              Done

            case res @CountingDown(count) =>
              res.copy(count - 1)

            case Done =>
              Done
          }
        }
      }
    }
  }
{% endhighlight %}
After a long time spent debugging the decrement function, I saw what the problem was, it was this line:

 case CountingDown(1) =>
    // last latch
    latchSignal.complete(())
    Done

I had totally forgotten that we were dealing with IO values, so to fulfil the `update` method function signature `A => A`, I added a `Done` after the `latchSignal.complete(())`. But that method isn't going to be run because it's an IO value.

The IO runtime won't run it because it wasn't returned, the IO computation as expected was just a description of computation and not the execution of the actual computation.

Now, after figuring this out, here was my next implementation for the decrement function:
{% highlight scala %}
override def decrement(): IO[Unit] = {
  latchState.updateAndGet {
    case CountingDown(1) | Done =>
      // last latch
      Done

    case res @CountingDown(count) =>
      res.copy(count - 1)
  }.flatMap {
    case Done =>
      // used attempt because of multiple calls to attempt
      latchSignal.complete(()).attempt.void
    case _ => IO.unit
  }
}
{% endhighlight %}

This worked, but i realized that since the ticking clock was continuous, the `complete` method of the latch was being called multiple times, throwing an error.

Then I remembered there was the `modify` method on the `Ref` that enabled returning some other state `B` and using that method seemed to totally solve the problem as seen below:
{% highlight scala %}
object CountdownLatch {
    def apply(n: Int): IO[CountdownLatch] = {
      require(n > 0, "number of latches should be greater than 0")
      for {
        latchSignal <- Deferred[IO, Unit]
        latchState <- Ref[IO].of[LatchState](CountingDown(n))
      } yield new CountdownLatch {
        override def await(): IO[Unit] =
          latchSignal.get

        override def decrement(): IO[Unit] = {
          latchState.modify {
            case CountingDown(1) =>
                // last latch
                Done -> latchSignal.complete(())

            case res@CountingDown(count) =>
              res.copy(count - 1) -> IO.unit

            case Done =>
              Done -> IO.unit

          }.flatten
        }
      }
    }
  }
{% endhighlight %}

which coincidentally was similar to the answer in the book :)
