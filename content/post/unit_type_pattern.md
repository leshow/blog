---
title: "Unit Type Pattern"
date: 2018-09-08T12:28:03-04:00
draft: true
---

I always enjoy reading blogs about design patterns or tricks people have picked up writing Rust. This is one I've seen a few times but never read about.

I've been doing class assignments from the wonderful [Operating Systems cs140e](https://web.stanford.edu/class/cs140e/syllabus/). I highly recommend this class if you know Rust and would like to try writing some lower level code. The class involves building bits of an OS for the raspberry pi. It requires some hardware components, and it took me a while to source the parts they used in the class, particularly the CP2102 USB dongle (links at the end of the article).

## The problem

[Assignment 1](https://web.stanford.edu/class/cs140e/assignments/1-shell/) has multiple phases, one of which is to implement parts of the [Xmodem](https://en.wikipedia.org/wiki/XMODEM) protocol. Xmodem is is used for transferring data through serial interfaces. Although we haven't arrived there yet, I imagine it'll be used for transferring data from the pc to Pi through the CP2102 dongle.

This is the `Xmodem` struct as given at the start of the assignment:

```rust
pub struct Xmodem<R> {
    packet: u8,
    inner: R,
    started: bool,
    progress: ProgressFn
}

type ProgressFn = fn(Progress);

pub fn noop(_: Progress) {}
```

Notice `ProgressFn`, which here is a function pointer. After I completed the assignment, I came back to this. I wanted to show a running progress bar, but with a function pointer I had no ability to capture any outside variables in my `ProgressFn`. Or at least no way that I knew of. If I wanted to capture variables from another scope, I needed to refactor this to be a closure.

There's a lot of solutions on offer here. I could have just stuffed a `Box<dyn FnMut(ProgressFn)>` in the struct and been done with it, and while the Xmodem didn't have no_std, none of the other assignments did allocation and I didn't want to start here. So, feeling parsimonious, I decided against boxing altogether.

## First attempt

```rust
pub struct Xmodem<R, F>
where
    F: FnMut(Progress),
{
    packet: u8,
    inner: R,
    started: bool,
    progress: F
}
```

To me, this had a syllogism to it. I want to hold a closure, so I should bound my declaration by that. When you introduce bounds here, they propagate to the every impl block afterwards. See if you can spot the problem here:

```rust
impl<T: io::Read + io::Write, F: FnMut(Progress)> Xmodem<T, F> {
    pub fn new(inner: T) -> Self {
        Xmodem {
            packet: 1,
            started: false,
            inner,
            progress: progress::noop,
        }
    }

    pub fn new_with_progress(inner: T, f: F) -> Self {
        Xmodem {
            packet: 1,
            started: false,
            inner,
            progress: f,
        }
    }
    // and others
}
```

`Xmodem::new` is returning `progress::noop`, our function pointer. But it's type variable says that it must be a `Xmodem<T, F>` where F is some `FnMut(Progress)`. Substituting a `|_: Progress| {}` in it's place doesn't help either, it's still choosing some specific trait when there's a more general one in the type signature. Our type `F` claims to be valid for all `FnMut`, but we're giving it a specific closure/fn pointer. You may recognize this as being an existential type.

Here's the error:

```bash
error[E0308]: mismatched types
   --> src/lib.rs:143:23
    |
143 |             progress: progress::noop,
    |                       ^^^^^^^^^^^^^^ expected type parameter, found fn item
    |
    = note: expected type `F`
               found type `fn(progress::Progress) {progress::noop}`
```

## Break up new and new_with_progress

We can get the above to compile by breaking up the `new` method into it's own impl that's specific about the type it returns:

```rust
impl<T: io::Read + io::Write> Xmodem<T, fn(Progress)> {
    pub fn new(inner: T) -> Xmodem<T, fn(Progress)> {
        Xmodem {
            packet: 1,
            started: false,
            inner,
            progress: progress::noop,
        }
    }
}
```

However, when you actually go to use this code, it produces a compiler error that after an hour of googling, I have no idea how to satisfy:

```bash
error[E0284]: type annotations required: cannot resolve `<_ as std::ops::FnOnce<(progress::Progress,)>>::Output == ()`
  --> src/tests.rs:51:48
   |
51 |     let tx_thread = std::thread::spawn(move || Xmodem::transmit(&input[..], rx)); // <FnOnce<(Progress)>>::Output = ()
   |                                                ^^^^^^^^^^^^^^^^
   |
note: required by `<Xmodem<(), F>>::transmit`
  --> src/lib.rs:41:5
   |
41 | /     pub fn transmit<R, W>(data: R, to: W) -> io::Result<usize>
42 | |     where
43 | |         W: io::Read + io::Write,
44 | |         R: io::Read,
45 | |     {
46 | |         Xmodem::transmit_with_progress(data, to, progress::noop)
47 | |     }
   | |_____^
```

Even if this can be satisfied with an annotation, I'll have broken every callsite for `Xmodem` by needing to type information. I decided to go back and try something else.

## Pattern: unit type params

In the original implementation, `Xmodem` does something interesting with it's `inner` value. It's set to an unbounded type parameter `R`. I decided to take this approach with `F`. If anyone has written Haskell, this may look familiar. Data types like a binary tree aren't usually bounded by `Ord` in the declaration, but the functions that interact with it provide an interface to the data that is bounded. I think this is a good strategy if it's possible. Try to keep the actual type declaration as generic as possible. You can control the bounds by choosing which methods you make public. Only in the impl are the bounds introduced (and then only on methods).

The new `Xmodem` type:

```rust
pub struct Xmodem<R, F> {
    packet: u8,
    started: bool,
    inner: R,
    progress: F,
}
```

and for the implementation:

```rust
impl Xmodem<(), ()>
{
    pub fn transmit<R, W>(data: R, to: W) -> io::Result<usize>
    where
        W: io::Read + io::Write,
        R: io::Read,
    {
        Xmodem::transmit_with_progress(data, to, progress::noop)
    }
    pub fn transmit_with_progress<R, W, F>(mut data: R, to: W, f: F) -> io::Result<usize>
    where
        W: io::Read + io::Write,
        R: io::Read,
        F: FnMut(Progress)
    {
        let mut transmitter = Xmodem::new_with_progress(to, f);
        //...
    }
    // and others
}

impl<T: io::Read + io::Write> Xmodem<T, fn(Progress)> {
    pub fn new(inner: T) -> Xmodem<T, fn(Progress)> {
        Xmodem {
            packet: 1,
            started: false,
            inner,
            progress: progress::noop,
        }
    }
}
with_progress
```

If I had put `impl<F> Xmodem<(), F>`, I would either have to bound the entire `impl` block by `where F: FnMut(Progress)`, which gets me back to:

```bash
error[E0284]: type annotations required: cannot resolve `<_ as std::ops::FnOnce<(progress::Progress,)>>::Output == ()`
  --> src/tests.rs:51:48
   |
51 |     let tx_thread = std::thread::spawn(move || Xmodem::transmit(&input[..], rx)); // <FnOnce<(Progress)>>::Output = ()
```

Or leave it unbounded, which would cause a compilation error when calling `new_with_progress`:

```bash
106 |         let mut receiver = Xmodem::new_with_progress(from, f);
    |                            ^^^^^^^^^^^^^^^^^^^^^^^^^ expected an `FnMut<(progress::Progress,)>` closure, found `F`
```

Unit's type and value are both `()`. In this case, we're able to satisfy the type parameters in the impl declaration and still control the bounds on the methods. The catch here is of course, the methods `transmit` `transmit_with_progress` and a few others not mentioned here are now only valid when `Xmodem::<(), ()>`.

There's nothing prodigious about `()`. Any type will do. We could've written:

```rust
impl Xmodem<u8, u8> {
    pub fn transmit<R, W>(data: R, to: W) -> io::Result<usize>
    where
        W: io::Read + io::Write,
        R: io::Read,
    {
        Xmodem::transmit_with_progress(data, to, progress::noop)
    }
    // etc
}
```

And everything would work just fine. I've seen `()` used before and I think it creates less room for confusion though.

Note, on IRC, an improvement was suggested that makes the existential return a bit more explicit (thanks talchas). Using the `()` type param again on the `new` impl:

```rust
impl<T> Xmodem<T, ()>
where
    T: io::Read + io::Write,
{
    pub fn new(inner: T) -> Xmodem<T, impl FnMut(Progress)> {
        Xmodem {
            packet: 1,
            started: false,
            inner,
            progress: progress::noop,
        }
    }
}
```

## On Nightly

Thanks to talchas on IRC, who mentioned a solution that's available behind a feature flag on nightly. It's possible to declare a type as existential and circumvent having to pass `()` as a type parameter in the impl block.

```rust
#![feature(existential_type)]
existential type ProgressFn: FnMut();

struct Xmodem<F>(F);

impl Xmodem<ProgressFn> {
    fn new() -> Self {
        Xmodem(||())
    }
}
```

To me this seems closer to ideal. We have the language to tell the type system that ProgressFn is in existential and pass it as a type parameter directly.

## Thank you

Thanks for reading! The Rust community is wonderful, special thanks to those on the #rust IRC

### CP2102 dongle

Be careful when you source this part to get the 5Pin version and not the 6Pin one that seems fairly popular online.

USA: [cp2102](https://www.amazon.com/HiLetgo-CP2102-Converter-Adapter-Downloader/dp/B00LODGRV8/ref=sr_1_4?s=electronics&ie=UTF8&qid=1536424599&sr=1-4&keywords=cp2102)

Canada: [cp2102](https://www.amazon.ca/Gikfun-CP2102-Converter-Arduino-EK1110x2C/dp/B07DXDLPMZ/ref=sr_1_5?s=electronics&ie=UTF8&qid=1536424620&sr=1-5&keywords=cp2102&dpID=41qk9Czr2sL&preST=_SY300_QL70_&dpSrc=srch)
