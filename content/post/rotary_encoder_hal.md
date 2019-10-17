---
title: "Rotary Encoder in Embedded Rust"
date: 2019-10-16T20:31:51-04:00
draft: true
---

Recently, I've been trying to learn more about electronics and embedded development. Maybe I'm just tired of operating purely in the virtual, but there's something cool about being able to physically put together a circuit and push a button to make something happen. I went through the usual Arduino resources before seeing what Rust had to offer. I'm happy to report there's some really good material out there.

We have,

- [Discovery book](https://rust-embedded.github.io/discovery/index.html)
- [Rust Embedded book](https://docs.rust-embedded.org/book/intro/index.html)
- [Real Time For the Masses](https://japaric.github.io/rtfm5/book/en/preface.html)

If you're new to embedded (but not new to Rust), I'd recommend the Discovery book as your jumping off point. It runs through the basics of how to build an embedded program, how to debug it, how to step through with GDB, how to read a data sheet, etc. After that, I'd recommend the rust embedded book, it will show you how the ecosystem is put together.

Embedded in Rust seems to really be taking off lately, there is a [weekly driver initiative](https://github.com/rust-lang-nursery/embedded-wg/issues/39) which I hope this crate is good enough to be considered. I've been hanging out in the Rust embedded matrix room if anyone wants to get at me and give me some feedback, the code that I'll be showing is also published on [github](https://github.com/leshow/rotary-encoder-hal), though not yet on `crates.io`.

## Rust Embedded Ecosystem

It's possible to build drivers that are hardware agnostic due to the `embedded-hal` crate. It's a collection of traits and types that abstract over specific implementations. `rotary-encoder-hal` uses the [InputPin](https://docs.rs/embedded-hal/0.2.3/embedded_hal/digital/v2/trait.InputPin.html) trait, for example. I'll get into the details a bit more later.

If you're interested in how the ecosystem fits together, I recommend checking out this [chapter](https://docs.rust-embedded.org/book/start/registers.html) in the embedded book.

## What's a Rotary Encoder

It's a peripheral whose position is encoded as digital output. In other words: it's a knob that you can turn. One nice thing about rotary encoders is they spin infinitely. The way they work is pretty simple, inside the encoder there's two pins and a bunch of evenly spaced contacts, as you spin, the pins will touch the contacts in a defineable way to create a square wave from which you can determine the direction. I apologize ahead of time for the quality of my drawings.

![Encoder wheel 1](/rotary_encoder_hal/IMG_1.jpg)

Then if the encoder was turned,

![Encoder wheel 2](/rotary_encoder_hal/IMG_2.jpg)

The resulting square wave would look like (top wave is A, bottom is B),

![Square wave](/rotary_encoder_hal/IMG_square.jpg)

The key thing to notice here is that they are 90° out of step.

## Implementation

The implementation is fairly straightforward. If we consider the square waves from the previous section and those that naturally follow from turning the other direction, we can construct a truth table of the current state/old state to determine whether a turn occurred and which way.

| Current | Old |        Direction |
| ------- | :-: | ---------------: |
| 00      | 01  |        Clockwise |
| 01      | 11  |        Clockwise |
| 10      | 00  |        Clockwise |
| 11      | 10  |        Clockwise |
| 00      | 10  | CounterClockwise |
| 01      | 00  | CounterClockwise |
| 10      | 11  | CounterClockwise |
| 11      | 01  | CounterClockwise |

We can represent the complete state as a single `u8`, and shift right by 2 (>> 2) to 'move in' the current state.

This truth table is implemented using the `From` trait for `u8`

```rust
pub enum Direction {
    Clockwise,
    CounterClockwise,
    None,
}

impl From<u8> for Direction {
    fn from(s: u8) -> Self {
        match s {
            0b0001 | 0b0111 | 0b1000 | 0b1110 => Direction::Clockwise,
            0b0010 | 0b0100 | 0b1011 | 0b1101 => Direction::CounterClockwise,
            _ => Direction::None,
        }
    }
}
```

I've got a `Rotary` struct that holds both pins and state.

```rust
pub struct Rotary<A, B> {
    pin_a: A,
    pin_b: B,
    state: u8,
}
```

Now for the `embedded-hal` part. How can we say that `A` and `B` are gpio pins? `embedded-hal` helpfully exposes an `InputPin` trait, that has methods `is_low()` and `is_high()` for reading the state of the pins. Using this, we can constrain the implementation of `Rotary` to take two generic pins.

```rust
impl<A, B> Rotary<A, B>
where
    A: InputPin,
    B: InputPin,
{
    pub fn new(pin_a: A, pin_b: B) -> Self {
        Self {
            pin_a,
            pin_b,
            state: 0u8,
        }
    }
    //...
}
```

`Rotary` exposes an `update()` method, that when called reads the value for `pin_a` and `pin_b` into `state` using bitwise 'or'.

```rust
pub fn update(&mut self) -> Result<Direction, Either<A::Error, B::Error>> {
    // use mask to get previous state value
    let mut s = self.state & 0b11;
    // move in the new state
    if self.pin_a.is_low().map_err(Either::Left)? {
        s |= 0b100;
    }
    if self.pin_b.is_low().map_err(Either::Right)? {
        s |= 0b1000;
    }
    // shift new to old
    self.state = s >> 2;
    // and here we use the From<u8> implementation above to return a Direction
    Ok(s.into())
}
```

I hope you liked that little foray into the world of Rust embedded! I'm a beginner in this space myself, so if anything seems out of sorts, let me know. Thanks for reading.
