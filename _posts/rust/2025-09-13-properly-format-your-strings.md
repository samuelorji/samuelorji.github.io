---
layout: post
title:  "Properly format your strings in rust"
date:   "2025-09-12 07:37:00 +0100"
tags: rust string
---

Rust, like any other language has string formatting/interpolation abilities similar to C's `printf` or `fprintf` functions, provided by the [fmt module](https://doc.rust-lang.org/std/fmt/). But it's not that simple as we will see below.

As an example, Here's an example of [filling and aligning](https://doc.rust-lang.org/alloc/fmt/index.html#fillalignment) text.

```rust
println!("|{:-<9}|", "Hello")
```
By using this format `{:-<9}`, we're saying that we want the string "Hello" to be left aligned to a width of 9. From the docs, if the length of the string to be formatted (hello) is less than the width (9), then the remaining spaces should be filled with the filler '-'

Here's the output:
```bash
|Hello----|
```

It all looks good and everything works as expected, until we run into a subtle issue. 

Here's an example of printing or "displaying" an enum in rust.

```rust
use std::fmt;
use std::fmt::{format, Display};

fn main(){
    enum IPVersion {
        IPV4,
        IPV6
    }
    impl Display for IPVersion {
        fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
            match self {
                IPVersion::IPV4 => write!(f, "IPV4"),
                IPVersion::IPV6 => write!(f, "IPV6"),
            }
        }
    }
    let ipv4 : IPVersion = IPVersion::IPV4;
    println!("|{:-<9}|", ipv4)
}
```

We defined our enum, implemented the `Display` trait for it so we can use it in the `println` statement with the same formatting as before, but for some weird reason, here's the output.

```bash
|IPV4|
```

What? where's the filler, why didn't that format as expected ?

The answer lies in a not so obvious statement in the `fmt` module

> Note that alignment might not be implemented by some types.

That initially didn't make much sense to me as there is no `Alignment` trait that needs to be implemented, why did the string (or str in rust)  `Hello` work, but an enum that had its `Display` trait implemented as a string didn't.


To further understand why, I decided to take a look at the implementation of `Display` for `str` and [here it is](https://github.com/rust-lang/rust/blob/52618eb338609df44978b0ca4451ab7941fd1c7a/library/core/src/fmt/mod.rs#L2751).

```rust
impl Display for str {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result {
        f.pad(self)
    }
}
```

The implementation caught my attention.

```rust
f.pad(self)
```
Why the call to `pad`, and I checked the [docs for `pad`](https://doc.rust-lang.org/core/fmt/struct.Formatter.html#method.pad) and here's an excerpt. 

> Takes a string slice and emits it to the internal buffer after applying the relevant formatting flags specified.

So, as expected, I decided to change my `Display` implementation.

```rust
impl Display for IPVersion {
        fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
            let rep = match self {
                IPVersion::IPV4 => "IPV4",
                IPVersion::IPV6 => "IPV6"
            };
            f.pad(rep)
        }
    }
```

and that seemed to work, here's my output.

```bash
|IPV4-----|
```
----------
So what is this `fmt::Formatter` and  why did calling `pad` on it seem to work.

### Why it works.

Very simply put, when we call `println` with the formatting options like `{:-<9}` in our case, it creates a [formatter](https://github.com/rust-lang/rust/blob/52618eb338609df44978b0ca4451ab7941fd1c7a/library/core/src/fmt/mod.rs#L1451) `f` of type `fmt::Formatter` with our formatting options and passes it as an argument to our `fmt` method in our `Display` trait implementation. [Here's an example of how that is called](https://github.com/rust-lang/rust/blob/52618eb338609df44978b0ca4451ab7941fd1c7a/library/alloc/src/string.rs#L2820C9-L2820C26). 

**It is weirdly now up to us to use the formatting options via the `pad` function or some other method I don't know about.**

If we look back at our first implementation that didn't get formatted correctly, we had our formatter `f`, but we simply wrote to it via the `write!` macro and it didn't apply our formatting flags. 

But by now explicitly applying our formatting flags from the formatter via the `pad` method. It seemed to work.

Now, does it make sense to have to explicitly call `pad` ourselves ?. Shouldn't that be implicitly called or added by the compiler ?.

That's for the really smart folks over at rustomania to figure out.



### Deeper look (Demo)
If you're still here, great, let's try to go a little deeper to see what's going on.

Let's add a `println` statement to see what the formatting options are when we try to format our string with `{:-<9}`. 

```rust
impl Display for IPVersion {
        fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
            // try to print formatting options that were supplied
            println!("Formatting Options, alignment: {:?}, Fill: {:?}, width: {:?}", f.align(), f.fill(), f.width());
            let rep = match self {
                IPVersion::IPV4 => "IPV4",
                IPVersion::IPV6 => "IPV6"
            };
            f.pad(rep)
        }
    }
```

We get this output.

```bash
Formatting Options, alignment: Some(Left), Fill: '-', width: Some(9)
```

This makes sense, as we want our text "Left" aligned, with a width of "9" the empty spaces filled with "-". 

So it proves our point that the formatting options are gotten from the `println!` statement and passed to the formatter `f` that's used in the the `fmt` method of the `Display` trait. 

You can try using a new formatter as is done when `to_string` is called [here](https://github.com/rust-lang/rust/blob/52618eb338609df44978b0ca4451ab7941fd1c7a/library/alloc/src/string.rs#L2817) and you'll see that formatting will not work as expected as your new formatter doesn't have any formatting options.

