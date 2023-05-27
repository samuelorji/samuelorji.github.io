---
layout: post
title:  "Essential Effects 04: Granular Parallelism (Fibers)"
date:   2021-04-09 12:37:00 +0100
tags: cats fp functional-programming scala essential-effects
---


### Forking, Joining and cancelling an effect

As explained by the use of `ParallelIO` which is more or less the effectful form of Futures.  
It thus means that there has to be a way to run effects concurrently and/or in parallel.

It means there must be a wy to run our effect on another thread, wait for it to finish, and  
continue execution.

To explain this, we will try to rewrite the `parMapN` function that comes with cats effect, but we''ll call ours `myParMapN` with the following signature:

`def myParMapN[A, B, C](ia: IO[A], ib: IO[B])(f: (A, B) => C): IO[C] = ???`

which should do the following.

- start both ia and ib computations so they run concurrently (i.e fork them)
- wait for their results
- cancel the other effect if one fails
- combine the results with the f function

To cancel, wait or join, we need some sort of reference to the concurrent computation, in regular programming, we will easily hold reference to a thread, but in Cats Effect, instead of a thread, we have a fiber.

#### Fibers

Fibers are like green threads. When using a for comprehension, the computations are sequenced, so each computation is only going to start when the previous one successfully completes.

Instead of waiting for the computation to finish, we could `fork` that computation to work on  
another thread.  
Example:
```scala
for {
 e1 <- effect1.start
 e2 <- effect2
} yield ()
```



Now, by calling start on effect, we fork the effect, and it starts to run on another thread (if applicable) and effect2 doesn't need to wait until e2 finishes.

The method signature for the start method is:

def start(implicit cs: ContextShift[IO]): IO[Fiber[IO, A]]

> If you start an effect, the current execution is “forked”: the started effect’s  
> execution is shifted to a different thread

The `start` method takes an implicit parameter `ContextShift[IO]` which we can think of as an Execution Context that is responsible for scheduling tasks on other threads.

It's important to know that it returns a fiber in an IO. If it just returned a fiber, this would mean that the original fiber had already started.

> It returns a Fiber inside an IO because if it instead produced, directly, a Fiber, that  
> would mean our original IO is running right now, but in reality it isn’t. The source  
> IO only executes when we explicitly run it, so we need to delay access to this fiber—by wrapping it in an effect—until the source IO is executed.

Similar to threads, Fibers are low level computing mechanisms for concurrent control, and it's advisable to use higher level abstractions and operations.

### Joining fibers

Similar to threads, fibers can be joined too. Calling join on a fiber will block the calling  
thread (thread that called 'join') until the fibers returns a result.

Here's an example:
```scala
def printMessagefromThread(msg: String) = {
 println(s"[${Thread.currentThread().getName}]: $msg")
}
val prog =
 for {
  _ <- IO.unit
  _ <- IO(printMessagefromThread("Before fork"))
  fiba <- (IO.sleep(1 second) >> IO(printMessagefromThread("I am running on another thread"))).start
  _ <- IO(printMessagefromThread("After fork"))
  _ <- fiba.join
  _ <- IO(printMessagefromThread("After Join"))
 } yield ()
 ```

which prints out:

[ioapp-compute-0]: Before fork
[ioapp-compute-0]: After fork
[ioapp-compute-1]: I am running on another thread
[ioapp-compute-1]: After Join

Analysing the result we see that `After Fork` was printed before the forked fiber printed anything. This thus means that it didn't wait for the fiber to finish execution. On the other hand, we see that after join was only called after the call to `Fiber#join` completed. We can prove this by even extending the fiber sleep time and we are sure that it won't be called until that fiber is completed.

It's worth noting that when we join a Fiber, execution continues on the thread the Fiber  
was running on, that explains why `After Join` was printed on another thread other than the one that printed `After Fork`

Now, we can initially write our `myParMapN` like this :
```scala
def myParMapN[A,B,C](ia : IO[A], ib: IO[B])(f : (A,B) => C) : IO[C] = {
  for {
   fib1 <- ia.start
   fib2 <- ib.start
   a <- fib1.join
   b <- fib2.join
  } yield f(a,b)
}
```

#### Cancelling a Fiber

We can cancel fibers. Imagine spinning up two CPU intensive computations in parallel, and we want the ability to cancel any one of them. We can easily do that with fibers by cancelling them.

Fibers can be cancelled by calling cancel on the fiber (Fiber#cancel). Cancellation is Idempotent in the sense that calling cancel on an already canceled fiber is the same as doing it once.

But calling join on a cancelled fiber. the join will never finish because no result will ever be produced which is opposite to regular threads that just return if we call join on an already terminated thread.

A naiive way to cancel can be:
```scala
def myParMapN[A,B,C](ia : IO[A], ib: IO[B])(f : (A,B) => C) : IO[C] = {
  for {
    fib1 <- ia.start
    fib2 <- ib.start
    a  <- fib1.join.onError({ case _ => fib2.cancel })
    b <- fib2.join.onError({ case _ => fib1.cancel })
  } yield f(a,b)
}
```
Where we register an `onError` handler on the IO result produced by the fiber

Registering an onErrorHandler is itself an effect, so it will only be registered after the fiber being joined has completed its join.

> The issue is that registering an onError handler is itself an effect, so in the code  
> above the handler would only be registered if we couple it to the result of fib1.join. But if we do that, then we won’t be registering the onError handler  
> with the result of fib2 until after fib1 has actually finished

So, this means that the onError handler of fib2 is not registered until fib1 has completed.

So, if fib1 fails, the error handler on fib2 isn't registered to cancel fib1

It's quite hard to directly implement cancellation using fibers, so we instead use other  
higher-order abstractions to handle cancellation. One of them is `IO#race` where you supply two computation values represented as `IO` values and it returns the first completed effect, either the first or second represented as an Either

> Run two IO tasks concurrently, and return the first to finish, either in success or error. The loser of the race is canceled. The two tasks are executed in parallel if asynchronous, the winner being the first that signals a result.

Looking at this example:
```scala
val task: IO[Unit] = IO.sleep(100 millis) *> IO(println("task"))
val timeout: IO[Unit] = IO.sleep(200 millis) *> IO(println("timeout"))
val program = for {
      done <- IO.race(task, timeout)
      _ <- done match {
        case Left(_) => IO(println("task: won"))
        case Right(_) => IO(println("timeout: won"))
      }
    } yield ExitCode.Success
```

We see this when we run the program:

task
task: won

We see that the other effect that should have printed `timeout` didn't print it out to the console because it was cancelled by the race method.

IO#race is built upon a simpler abstraction IO#racePair which returns the winning effect and the fiber of the other effect.

> IO#racePair: Run two IO tasks concurrently, and returns a pair containing both the winner's successful value and the loser represented as a still-unfinished task.  
> If the first task completes in error, then the result will complete in error, the other task being canceled.
