---
layout: post
title:  "Essential Effects 05: Contexts and Thread Pools"
date:   2021-04-10 12:37:00 +0100
tags: cats fp functional-programming scala essential-effects
---

#### CPU vs IO Bound work

##### IO Bound work

IO operations typically involves reading or writing to files, databases, sockets e.t.c. Most of these times, waiting is involved as sometimes data isn't yet available.  
In these scenarios, the CPU will have to wait until the underlying hardware has delivered the data since it is not immediately available.

Any IO operation that requires reading or writing using anything that is not stored in RAM  
will cause something called IOwait, which is basically a system call that tells the CPU to pause the execution og the current thread until data is available or data has been successfully transmitted. This way. the CPU can easily pause that thread and move on to do other things.

During the process of IOwait, the OS is free to pick up another thread and execute that thread.  

If you have too few threads available for doing I/O work, the performance of your application will suffer as they can’t even get started until some other I/O operation is completed and that thread goes back to the thread pool.

##### CPU Bound Work

Let's imagine we want to do something like blur an image that is in memory. Because that image is already in memory and the CPU doesn't have to wait for data and there won't be calls to IOwait. Now, this means that the OS won't allow other threads to run as this process doesn't pause. Well, this is not technically true, the OS normally and automatically switches  
execution of threads even without a call to IOwait. This is to give other threads a fair chance to run.

But in our case of the CPU intensive work, there won't be as many pauses or context switches compared to IO bound work because there were no explicit IOwait calls.

Now, what this means is that if we have a thread pool that mixes CPU and IO work, the IO work will mostly depend on the OS to automatically give it execution time as the CPU intensive work won't willingly yield execution time compared to an all IO work where each thread willingly yields execution time until data is available for processing.

That's why it's quite important to create separate thread pools for different kinds of work.

[inspired by this wonderful post](https://www.hellsoft.se/understanding-cpu-and-i-o-bound-for-asynchronous-operations/)

### Multiple Thread pools.

By default, the `IOApp` provides us with a fixed pool thread executor where the  
number of threads is set to the number of available CPUs  
`Runtime.getRuntime.availableProcessors()`

> What do we do if our pool has at most n threads, but all those threads are  
> blocked? If that happens, we can’t use any available cores to do CPU-bound work.  
> To ensure our programs make progress—ensuring work proceeds when I/O-bound work is blocked—we’ll isolate the CPU-bound work from any I/O-bound tasks by having separate pools.

The Cats effect Library supports this pattern by encouraging separate contexts.

- CPU- bound work (using a fixed thread pool)
- IO bound work (using an unbounded thread pool so blocked threads merely take memory and don't prevent other tasks from running)

For IO bound tasks, cats effect provides a small wrapper around a Cached Thread pool execution context called a Blocker for execution.
```scala
  val prog =  Blocker.apply[IO].use{ blocker =>
    withBlocker(blocker).as(ExitCode.Success)
  }

  def withBlocker(blocker : Blocker) : IO[Unit] = {
    for {
      _ <- IO("on IOapp threadpool").debug
      _ <- blocker.blockOn(IO("on blocker").debug)
      _ <- IO("<---- Thread I'm on").debug
    } yield ()
  }
```

Here's what we see on the console

```bash
[ioapp-compute-0] on IOapp threadpool
[cats-effect-blocker-0] on blocker
[ioapp-compute-1] <---- Thread I am on
```

We can clearly see that the blocking effect was executed on another thread pool  
`cats-effect-blocker-*` pool other than the one the previous effect ran on `ioapp-compute`

#### Finer Grained Control of Contexts

What if we want to control the contexts used to run our effects and easily switch between contexts.

Let's take the ticking clock example we designed a while back , It won't make sense if this ws made to execute continuosly on one thread and hoarding that thread, thus making that thread unavailable for other effects to use and reducing the amount of work our application can perform within a time frame.

> To ensure a recursive loop doesn’t steal a thread and never give it back, we’d like  
> to be able to declare, as an effect itself, “reschedule the remainder of the  
> computation”. Not only would this resume the computation on (potentially)  
> another thread when the resumption is executed by the context, but it then allows other scheduled effects to re-use the previous thread. In other words, the current effect is “suspended” and sent “to the back of the line”, which prevents other effects from being “starved” of a thread.

We can insert asynchronous boundaries and control to an extent the threads within a context that we want our effects to run using the IO.shift method If we run this code:

```scala
object Shifting extends IOApp {
 def run(args: List[String]): IO[ExitCode] =
  for {
   _ <- IO("one").debug
   _ <- IO.shift
   _ <- IO("two").debug
   _ <- IO.shift
   _ <- IO("three").debug
  } yield ExitCode.Success
}
```


We get
```bash
[ioapp-compute-0] one
[ioapp-compute-1] two
[ioapp-compute-2] three
```

It's obvious that for every shift, the next effect runs in a different thread in the same context.  
This is what inserting an async boundary means, where the next effect is made to run on another thread

The IO.sleep method does this because if blocked the current thread for the duration of the sleep, we'd be preventing that thread from being used by other effects.

> Cats Effect inserts an async boundary at runtime every 512 flatMap calls. This is a kind of fail-safe—if you forget to add a boundary yourself, the library will ensure that a composed effect can’t re-use the same thread for very long

The IO.shift is overloaded as it can also take an execution context as a parameter, so it's possible to switch effects to a different execution context rather than a different thread.
```scala
 val ec = ExecutionContext.Implicits.global
  val program = for {
    _ <- IO.shift(ec)
    _ <- IO(println(s"I am running on ${Thread.currentThread().getName}"))
  } yield ()
```
which prints out:

`I am running on scala-execution-context-global-12`
