---
layout: post
title:  "Coordinating threads with Condvar in Rust"
date:   "2025-09-11 07:37:00 +0100"
tags: rust threads
---

I recently worked on a rust project where I had to coordinate state between two threads, and with my Scala background, it wasn't the easiest to implement in rust which does things a little differently.

I decided to write this to help rust developers get a good intuition of what [Condvar](https://doc.rust-lang.org/std/sync/struct.Condvar.html) is, how it works and why it can be used for state coordination.

To drive our point home, we will be designing a very simple cli application that starts and prints a counter on a thread, and takes input from the main thread to either pause or resume that counter. We will start from a basic inefficient solution and then introduce how using a Condvar will make our system a little more efficient.

> The solution is pretty straight forward. We have some atomic variable that's shared between the two threads that will serve as a signal to pause or resume counting, in rust, shared atomic variable immediately implies `Arc<Mutex<T>>` which we will be using. 

Let's start by defining our mutex that will be used by both threads
```rust
use std::io;
use std::sync::{Arc, Mutex};
fn main() {
    // signal to be shared by both threads
    let signal : Arc<Mutex<bool>> = Arc::new(Mutex::new(false));
    // cloned signal that will be used by the main thread
    let input_arc = Arc::clone(&signal);
    // cloned signal that will be used by the counter thread
    let counter_arc = Arc::clone(&signal);
}

```

Now, let's see our counter thread:
```rust
let counter_thread = std::thread::spawn(move || {
        let mut count = 0;
        loop {
            // check the mutex if program should be paused
            let paused : MutexGuard<bool>  = counter_arc.lock().unwrap();
            if *paused {
                println!("Program is paused")
            } else {
                count += 1;
                println!("Counter: {}", &count);
            }

            drop(paused);

            // a thread sleep so we can slow down terminal output
            std::thread::sleep(std::time::Duration::from_secs(1));
        }
    });

```
In the code snippet above, we simply start a thread and define a local variable called count. In a loop, we first try to acquire / lock the mutex, checking the value. If it's true (paused), we simply print "Program is Paused", if not, we increment the counter and print it to the console. We also drop the mutex so it can be used/acquired by other threads.

> We've added a thread::sleep so we can see slow down the counter and see it incrementing each second, 

Now, let's write code in our main thread that will listen to input from the terminal to either pause or resume the counter above.

```rust

loop {
        let mut user_input = String::new();
        io::stdin().read_line(&mut user_input).unwrap();
        match user_input.trim() {
            "p" => {
                let mut pause: MutexGuard<bool> = input_arc.lock().unwrap();
                *pause = true;
                drop(pause);
                println!("Counter Paused")
            },
            "r" => {
                let mut pause: MutexGuard<bool> = input_arc.lock().unwrap();
                *pause = false;
                drop(pause);
                println!("Counter Resumed")
            },
            x =>  {
                println!("unknown command: '{}'", x)
            }
        }
    }

```

Here, we basically wrap getting user input in a loop and acting on the user input, if the user types 'p', we try to pause the counter by acquiring the lock on the mutex and setting its value to true. If the user types 'r', we resume the counter by acquiring the lock on the mutex and setting it to false. 
<details markdown="1">

<summary markdown="span">Here's the full code</summary>

```rust
use std::io;
use std::sync::{Arc, Mutex, MutexGuard};
fn main() {
    let signal: Arc<Mutex<bool>> = Arc::new(Mutex::new(false));
    let input_arc = Arc::clone(&signal);
    let counter_arc = Arc::clone(&signal);

    // thread that handles counting and printing to stdout
    let counter_thread = std::thread::spawn(move || {
        let mut count = 0;
        loop {
            // check the mutex if program should be paused
            let paused: MutexGuard<bool> = counter_arc.lock().unwrap();
            if *paused {
                println!("Program is paused")
            } else {
                count += 1;
                println!("Counter: {}", &count);
            }

            drop(paused);

            // a thread sleep so we can slow down terminal output
            std::thread::sleep(std::time::Duration::from_secs(1));
        }
    });


    // main thread that takes user input to either pause or resume the counter
    loop {
        let mut user_input = String::new();
        io::stdin().read_line(&mut user_input).unwrap();
        match user_input.trim() {
            "p" => {
                let mut pause: MutexGuard<bool> = input_arc.lock().unwrap();
                *pause = true;
                drop(pause);
                println!("Counter Paused")
            },
            "r" => {
                let mut pause: MutexGuard<bool> = input_arc.lock().unwrap();
                *pause = false;
                drop(pause);
                println!("Counter Resumed")
            },
            x =>  {
                println!("unknown command: '{}'", x)
            }
        }
    }
}

```

</details>

If we run this code with `cargo run`, wait a little bit, try to pause by typing 'p', and resume by typing 'r'. We see that that the program works as expected?.

## But ..... 

When you try to pause with 'p', you may notice that the program keeps printing "Program is paused" which is kinda inefficient.

Here's an example of an output from my machine:

<details markdown="1">

<summary markdown="span">Console Output</summary>

```bash
Counter: 1
Counter: 2
Counter: 3
Counter: 4
Counter: 5
Counter: 6
p
Counter Paused
Program is paused
Program is paused
Program is paused
Program is paused
Program is paused
Program is paused
Program is paused
Program is paused
r
Counter Resumed
Counter: 7
Counter: 8
Counter: 9
Counter: 10
Counter: 1
```
</details>

When the program is paused, we see that the thread just keeps going on and on, wasting 'precious' cpu cycles. 

Now, we can remove that code that prints to the console, and just pretend like nothing is happening, but deep down, we know that we just have a thread that's actively doing nothing and taking resources from other threads (if they exist). There must be a better way to do this where we put the counter thread to sleep and wake it only signalled by user input from another thread.

If only we had something similar to [wait](https://docs.oracle.com/javase/8/docs/api/java/lang/Object.html#wait--) in Java.

## Enter `Condvar`.

`Condvar` or "Conditional Variable" represents the ability to block a thread such that it consumes no CPU time while waiting for an event to occur. [See more here](https://doc.rust-lang.org/std/sync/struct.Condvar.html)

This looks like something that will work for us, we will block the counter thread, preventing it from consuming CPU until the user presses 'r' to resume. Let's see what that will look like.

First, we define our signal that now includes a `Condvar` and use it in the counter thread.

```rust
use std::io;
use std::sync::{Arc, Condvar, Mutex, MutexGuard};
fn main() {
    let signal: Arc<(Mutex<bool>, Condvar)> = Arc::new((Mutex::new(false), Condvar::new()));
    let input_arc = Arc::clone(&signal);
    let counter_arc = Arc::clone(&signal);

    //thread that handles counting and printing to stdout
    let counter_thread = std::thread::spawn(move || {
        let mut count = 0;
        loop {
            // check that the program is not paused
            let (lock, cvar) = &*counter_arc;

            // check the mutex if program is paused
            let mut paused: MutexGuard<bool> = lock.lock().unwrap();
            if *paused {
                println!("Program is paused");
                // block / pause this thread until someone calls cvar.notify() or cvar.notify_all() on the cvar
                paused = cvar.wait(paused).unwrap();
            } else {
                count += 1;
                println!("Counter: {}", &count);
            }

            drop(paused);

            // a thread sleep so we can slow down terminal output
            std::thread::sleep(std::time::Duration::from_secs(1));
        }
    });
}
```

Not much has changed, instead of our signal just containing a mutex, it's now a tuple that contains a mutex and a Condvar. 

In the counter loop, we do the same as before, we acquire the mutex, check the boolean value to confirm if the program is paused or not.

The new thing we introduced is this line:

```rust
paused = cvar.wait(paused).unwrap();
```

Now, this bit is crucial to understand. First, we call `wait` on the `Condvar`, passing the mutex (`paused`) as an argument. What happens here is that calling `wait` blocks or pauses the thread (counter thread) and in the process, releases the lock on the mutex that was passed as an argument (`paused`). 

So now, we have a blocked/paused thread and a free/unlocked mutex. The return value of the `wait` method is the mutex that was passed as an argument which represents the acquired state of the mutex once the thread wakes up.

 Here's an excerpt from the `wait` method on Condvar

> Blocks the current thread until this condition variable receives a notification.
This function will atomically unlock the mutex specified (represented by guard) and block the current thread. This means that any calls to notify_one or notify_all which happen logically after the mutex is unlocked are candidates to wake this thread up. When this function call returns, the lock specified will have been re-acquired.

It's important to understand why the mutex has to be released, because if the mutex is not released, then we have a situation where the blocked thread holds on to the mutex, preventing any other thread from using it to signal anything, resulting in a deadlock. 

Now let's see the main thread's input loop:

```rust
loop {
        let mut user_input = String::new();
        io::stdin().read_line(&mut user_input).unwrap();
        match user_input.trim() {
            "p" => {
                let (lock, _) = &*input_arc;
                let mut is_paused = lock.lock().unwrap();
                *is_paused = true;
                drop(is_paused);
                println!("Counter Paused")
            },
            "r" => {
                let (lock, cvar) = &*input_arc;
                let mut is_paused = lock.lock().unwrap();
                *is_paused = false;
                drop(is_paused);
                // notify / wake up whoever is waiting on this conditional variable
                cvar.notify_one();
                println!("Counter Resumed")
            },
            x => {
                println!("unknown command: '{}'", x)
            }
        }
    }
```
The only major thing that has changed here is in the 'r' resume block:

```rust
cvar.notify_one();
```

> This basically says, "Hey, to whoever that is 'waiting' on this variable, WAKE UP !!!!". 

Once `notify_one` is called, the counter thread that was blocked by the call to `wait` on the condvar will wake up, acquire the mutex, see if the signal is now set to false and then resume its operation. 

Here's my console output now.

<details markdown="1">

<summary markdown="span"> Console Output</summary>


```bash
Counter: 1
Counter: 2
Counter: 3
Counter: 4
Counter: 5
Counter: 6
p
Counter Paused
Program is paused
r
Counter Resumed
Counter: 7
Counter: 8
Counter: 9
Counter: 10
p
Counter Paused
Program is paused
r
Counter Resumed
Counter: 11
Counter: 12
```
</details>

Now we see that when the program is paused, the thread actually stops work and is blocked until the user inputs 'r' to wake the thread and continue counting. 


### Condvar Gotchas.

#### 1. Spurious Wakeups:
Condvars are susceptible to spurious wakeups, which is when the thread blocked by the call to `wait` wakes up unexpectedly (without being notified), [see more here](https://en.wikipedia.org/wiki/Spurious_wakeup). 

Here's an excerpt from the `wait` method.

> Note that this function is susceptible to spurious wakeups. Condition variables normally have a boolean predicate associated with them, and the predicate must always be checked each time this function returns to protect against spurious wakeups.

Looking at our code where we call the `wait` method, we can see where this may be an issue 

```rust
let mut paused: MutexGuard<bool> = lock.lock().unwrap();
if *paused {
    println!("Program is paused");
    paused = cvar.wait(paused).unwrap();
}
```

if we have a spurious wakeup, our thread will wakeup without being notified by the `notify_one()` method in the main thread, causing it to keep executing whereas it should be paused.

Here's an illustration:

Imagine I have a piece of code that should only be run once the thread wakes up and the counter resumes:

```rust
fn run_on_thread_wakeup() {
    // do something expensive here
    println!("Hi, I'm about to resume counting")
}
```
and we add this code at the end of the the if block where we expect the program to continue from once the thread wakes up

```rust
let mut paused: MutexGuard<bool> = lock.lock().unwrap();
if *paused {
    println!("Program is paused");
    paused = cvar.wait(paused).unwrap();
    run_on_thread_wakeup()
}
```
If our thread spuriously wakes up and our pause signal hasn't been set to false by user input in the main thread, we call the `run_on_thread_wakeup` despite the fact the we are paused and shouldn't call the function yet. Nothing guards/checks that we're still paused leading to the invocation of the `run_on_thread_wakeup` function which is a false positive.

What we want instead is to guard against spurious wakeups by constantly checking that we're still paused (mutex is set to true) and only continue execution if the mutex/signal is set to false when we wake up like so:

```rust
let mut paused: MutexGuard<bool> = lock.lock().unwrap();
if *paused {
    println!("Program is paused");
    while *paused {
        // Whenever the thread wakes up, don't just continue execution, check and ensure that the signal is set to false.
        // only way we can break out of this while loop. Ensuring that even if the thread spuriously wakes up, we check our signal before moving forward.
        paused = cvar.wait(paused).unwrap();
    }
    run_on_thread_wakeup()
}

```

<details markdown="1">

<summary markdown="span"> Full code with Spurious wakeup guard</summary>


```rust
use std::io;
use std::sync::{Arc, Condvar, Mutex, MutexGuard};
fn main() {
    let signal: Arc<(Mutex<bool>, Condvar)> = Arc::new((Mutex::new(false), Condvar::new()));
    let input_arc = Arc::clone(&signal);
    let counter_arc = Arc::clone(&signal);


    let counter_thread = std::thread::spawn(move || {
        let mut count = 0;
        loop {
            // check that the program is not paused
            let (lock, cvar) = &*counter_arc;

            // check the mutex if program is paused
            let mut paused = lock.lock().unwrap();
            if *paused {
                println!("Program is paused");
                while *paused {
                    paused = cvar.wait(paused).unwrap();
                }
            } else {
                count += 1;
                println!("Counter: {}", &count);
            }

            drop(paused);

            // a thread sleep so we can slow down terminal output
            std::thread::sleep(std::time::Duration::from_secs(1));
        }
    });


    // main thread that takes user input to either pause or resume the counter
    loop {
        let mut user_input = String::new();
        io::stdin().read_line(&mut user_input).unwrap();
        match user_input.trim() {
            "p" => {
                let (lock, _) = &*input_arc;
                let mut is_paused = lock.lock().unwrap();
                *is_paused = true;
                drop(is_paused);
                println!("Counter Paused")
            },
            "r" => {
                let (lock, cvar) = &*input_arc;
                let mut is_paused = lock.lock().unwrap();
                *is_paused = false;
                drop(is_paused);


                // notify / wakeup whoever is waiting on this conditional variable
                cvar.notify_one();
                println!("Counter Resumed")
            },
            x => {
                println!("unknown command: '{}'", x)
            }
        }
    }
}
```
</details>


> Fun fact, I tried to see if i could experience any spurious wakeups and it only happened once over a span of 3 days.


#### 2. Explicitly Dropping the Mutex:
I think this goes without saying, but ensure your mutex is only acquired when you need it and dropped immediately after use, if a mutex is held longer than needed, it can cause issue. This gets worse if your mutex is held in a loop like in our case. 

The mutex can be dropped explicitly by calling the `drop` method, or by introducing local scopes for the mutexes that drop them once the local scope is out of scope (pun intended).

These are pretty much the same:
```rust
let (lock, _) = &*input_arc;
let mut is_paused = lock.lock().unwrap();
*is_paused = true;
drop(is_paused);
```
and

```rust
let (lock, cvar) = &*input_arc;
{
    let mut is_paused = lock.lock().unwrap();
    *is_paused = false;
} // mutex is_paused dropped here
```


That's all for today, hope you enjoyed it :)