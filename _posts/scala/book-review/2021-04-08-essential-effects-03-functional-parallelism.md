---
layout: post
title:  "Essential Effects 03: Functional Parallelism"
date:   2021-04-08 12:37:00 +0100
tags: cats fp functional-programming scala essential-effects
---

### Parallelism

Futures which are higher kinded `F[_]` types support parallelism by scheduling work on multiple threads via a scala.concurrent.ExecutionContext. This thus means that we can make Futures run in sequence and in parallel.  
Let's see this example:

{% highlight scala %}
object Future2 extends App {

  implicit val ec = ExecutionContext.global

  def sleepFor(time: Long) = Thread.sleep(time)

  def hello = Future {
    sleepFor(1000)
    println(s"[${Thread.currentThread.getName}] Hello")
  }

  def world = Future {
    sleepFor(1000)
    println(s"[${Thread.currentThread.getName}] World")
  }

  val st1 = System.currentTimeMillis()
  val hw1: Future[Unit] = {
    for {
      _ <- hello
      _ <- world
    } yield ()
  }

  Await.ready(hw1, 5.seconds)
  println(s"h1 took ${(System.currentTimeMillis() - st1)} milliseconds")

  val st2 = System.currentTimeMillis()
  val hw2: Future[Unit] =
    (hello, world).mapN((_, _) => ())

  Await.ready(hw2, 5.seconds)
  println(s"h2 took ${(System.currentTimeMillis() - st2)} milliseconds")
}
{% endhighlight %}

The first result `h1` took over 2 seconds, while the second result just took over  
1 second. `h1` uses a for comprehension which ensures that the computation is sequenced and "World" is printed only after "Hello" is printed. But with `mapN`, the result is undeterministic as we more or less spawn two Futures at once.

We composed functions using `flatMap` and `mapN` and we saw that mapN enabled parallelism for the Future data type.

This demonstrates that for Future, flatMap and mapN have different effects with  
respect to parallelism.

Now, let's replace the Future data type with an IO:

{% highlight scala %}
object Future3 extends App {

  implicit val ec = ExecutionContext.global

  def sleepFor(time: Long) = Thread.sleep(time)

  def hello = IO {
    sleepFor(1000)
    println(s"[${Thread.currentThread.getName}] Hello")
  }

  def world = IO {
    sleepFor(1000)
    println(s"[${Thread.currentThread.getName}] World")
  }

  val st1 = System.currentTimeMillis()
  val hw1: IO[Unit] = {
    for {
      _ <- hello
      _ <- world
    } yield ()
  }

  Await.ready(hw1.unsafeToFuture(), 5.seconds)
  println(s"h1 took ${(System.currentTimeMillis() - st1)} milliseconds")

  val st2 = System.currentTimeMillis()
  val hw2: IO[Unit] =
    (hello, world).mapN((_, _) => ())

  Await.ready(hw2.unsafeToFuture(), 5.seconds)
  println(s"h2 took ${(System.currentTimeMillis() - st2)} milliseconds")
}
{% endhighlight %}

If we run the code, we'll see that both will take about the same time and will run on the same thread (main), this thus means that there was no parallelism involved whatsoever even with `mapN`.

> But note: it isn’t the case mapN for Future is implemented with parallelism but  
> flatMap is implemented as something sequential. The parallelism comes as a side effect—(pun intended)—of Future eagerly scheduling the computation, which happens before mapN itself is evaluated.

Now, since IO doesn't eagerly compute, mapN and flatMap have the same effect.

> IO doesn’t provide any support for the effect of parallelism! And this is by design,  
> because we want different effects to have different types, as per our Effect Pattern.

### Parallel IO

We have seen that the IO type we have been working with doesn't support parrallelism, it acts more or less like non parallel higher kinded types such as regular scala collections.

If IO doesn’t support parallelism, we need a new type that does. In cats.effect, this type is named IO.Par (Par for “parallel”).

Now, this IO.Par type should not have a monad instance because we do not want to be able to serialize the execution of multiple actions (we don't want the ability to sequence actions like flatMap), but we need an applicative instance to be able to compose independent IO.par values.

The IO.par type is defined as:
{% highlight scala %}
object IO {
  class Par[+A] { ??? } 
  object Par {
    def apply[A](ioa: IO[A]): Par[A] = ??? 
    def unwrap[A](pa: Par[A]): IO[A] = ??? 
  }
}
{% endhighlight %}


Where we can convert wrap an `IO[A]` to give an `IO.Par[A]` as well as unwrapping an `IO.par[A]` to an `IO[A]`.  
Here's a mock of what the applicative of IO.par looks like:

{% highlight scala %}
implicit def ap(implicit cs: ContextShift[IO]): Applicative[IO.Par] = {
 new Applicative[IO.Par] {
   def pure[A](a: A): IO.Par[A] = IO.Par(a)
   def map[A, B](pa: IO.Par[A])(f: A => B): IO.Par[B] = ???
   def product[A, B](pa: IO.Par[A], pb: IO.Par[B]): IO.Par[(A, B)] = ???
  }
}
{% endhighlight %}

- Where ContextShift[IO] is required to be able to switch computations to different  
    threads, which for the present can be thought of as something similiar to a scala.concurrent.ExecutionContext or thread pool.
- The implementation of product will ensure that pa and pb execute on different  
    threads, using cs

> It's quite rare that we will interact with the `IO.par` type, so it's possible to convert from IO to IO.par types and  
> back via the `Parallel[IO].parallel(a : IO[A])` or `Parallel[IO].sequential(a : IO.Par[A])`

Now, let us try our previous example, where we created IO values and used `mapN` on them. Let' see if we can achieve any sort of parallelism.

{% highlight scala %}
object ParallelPlay extends App {

  val global = ExecutionContext.Implicits.global

  // context shift used by parallel IO for scheduling tasks on different threads
  implicit val cs : ContextShift[IO] = IO.contextShift(global) 

  def sleepFor(time: Long) = Thread.sleep(time)

  def hello: IO[Unit] = IO {
    sleepFor(1000)
    println(s"[${Thread.currentThread.getName}] Hello")
  }

  def world: IO[Unit] = IO {
    sleepFor(1000)
    println(s"[${Thread.currentThread.getName}] World")
  }

  val parallelHello: IO.Par[Unit] = Parallel[IO].parallel(hello) // converting from regular IO to a parallel IO 
  val parallelWorld: IO.Par[Unit] = Parallel[IO].parallel(world) // converting from regular IO to a parallel IO 

  val startTime = System.currentTimeMillis()
  val parallelResult: IO.Par[Unit] = (parallelHello,parallelWorld).mapN((_, _) => {
    println(s"computation took ${(System.currentTimeMillis() - startTime)} milliseconds")
  })

  val sequentialIO = Parallel[IO].sequential(parallelResult) // turn parallel IO to sequential IO 

  sequentialIO.unsafeRunSync()
}
{% endhighlight %}

If we run this example, we see hello and world printed on different threads as well as the total time just over 1 second.

Voila, we've achieved parallelism via the IO.Par type.

Now, it's quite obvious that it's gonna be tedious in real world programming to continuously convert between `IO` and `IO.Par`, so we have a convenient method on the regular IO called `parMapN` which behind the scenes does the same job of calling Parallel.sequential on an IO.Par as shown in the above example.

Here's what it looks like:
{% highlight scala %}
(hello,world).parMapN((_, _) => {
    println(s"computation took ${(System.currentTimeMillis() - startTime)} milliseconds")
  })
{% endhighlight %}

This will have the same result of running both IO effects in parallel. It's important to note that it `parMapN` similar to `mapN` works for other tuple types apart from Tuple2.

#### parTraverse

simply the parallel version of traverse with type signature:

`F[A] => (A => G[B]) => G[F[B]]`

simple usage: doing parallel work on a sequence and having the results combined in another F type
