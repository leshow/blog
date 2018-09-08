---
title: "Unit Type Pattern"
date: 2018-09-08T12:28:03-04:00
draft: true
---

I always enjoy reading blogs design patterns or tricks people have picked up writing Rust. This is one I've seen a few times but never read about.

I've been doing class assignments from the wonderful [Operating Systems cs140e](https://web.stanford.edu/class/cs140e/syllabus/). I highly recommend this class if you know Rust and would like to try writing some lower level code. The class involves building an OS for the raspberry pi. It took me a while to source the parts they used in the class, particularly the CP2102 USB dongle (links at the end of the article).

## The problem

In assignment 1 of cs140e, they have you implement the [Xmodem](https://en.wikipedia.org/wiki/XMODEM) protocol for the pi. Xmodem is a file transfer protocol for transferring data through serial interfaces. I imagine in the context of the class it's meant for transferring data from the pc to Pi through the CP2102 usb dongle.

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

After `Xmodem` is implemented the next task is to use it in creating a cli tool, this tool can provide a `ProgressFn`, which here is just a function pointer. I wanted to show a running progress bar, but with just a function pointer I had no ability to capture any outside variables in my `ProgressFn`. In short, if I wanted to capture outside variables, I needed to refactor this to be a closure.

I could have just stuffed a `Box<dyn FnMut()>` in the struct and been done with it, but this was an operating systems class, and I have a feeling Xmodem will be running in some kind of no_std context in the future. So, feeling parsimonious, I decided against boxxing altogether.

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

To me, this seemed like the logical place to start. I should mention the supporting methods at this point, see if you can spot the error:

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

`Xmodem::new` is returning `progress::noop`, our function pointer. But it's type variables say that it must be a `Xmodem<T, F>` where F is some `FnMut(Progress)`. Substituting a `|_: Progress| {}` in it's place doesn't help either. The reality is that our type claims to be valid for all `FnMut`, but we're giving it a specific closure/fn pointer. You may recognize this as being an existential type.

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

We can get the above to compile by breaking up the new method into it's own impl that's specific about the type it returns:

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

Additionally, if this can be satisfied, I've broken every callsite for `Xmodem` by needing to add a type annotation. It seems another approach is necessary.

## Pattern: unit type params

`Xmodem` does something interesting with it's `inner` value. In the definition, it's set to an unbounded type parameter `R`. In it's method implementations, it sets the type parameter to `()` and then bounds the type on it's method. On a hunch, I decided to take this approach to `F`. As an aside, if anyone has written Haskell, leaving the bounds off your data type should look familiar. Data types like a binary tree aren't usually bounded by `Ord` in the declaration, but the functions that interact with it provide an interface to the data that is bounded.

```rust
pub struct Xmodem<R, F> {
    packet: u8,
    started: bool,
    inner: R,
    progress: F,
}
```

and then in the implementation:

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

Unit is an odd type. It's type and value are both `()`. In this case, we're able to satisfy the type arguments while providing methods that are parametric and

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

Here you just tell the type system that ProgressFn is in existential and pass it as a type parameter directly. This is super cool and says exactly what I am trying to express.

// syllogism
// felicitous

## CP2102 dongle

Be careful when you source this part to get the 5Pin version and not the 6Pin one that seems fairly popular online:

[cp2102 amazon USA](https://www.amazon.com/HiLetgo-CP2102-Converter-Adapter-Downloader/dp/B00LODGRV8/ref=sr_1_4?s=electronics&ie=UTF8&qid=1536424599&sr=1-4&keywords=cp2102)

[cp2102 amazon CA](https://www.amazon.ca/Gikfun-CP2102-Converter-Arduino-EK1110x2C/dp/B07DXDLPMZ/ref=sr_1_5?s=electronics&ie=UTF8&qid=1536424620&sr=1-5&keywords=cp2102&dpID=41qk9Czr2sL&preST=_SY300_QL70_&dpSrc=srch)

## Thank you

The Rust community is wonderful, and I'd like to thank the IRC regulars in particular, they helped me a great deal and suggested some of these solutions; talchas and dodo specifically for the nightly existential type workaround.
