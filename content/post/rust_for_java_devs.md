---
title: "Rust for Java Devs"
date: 2020-02-02T22:23:50-05:00
draft: true
---

I've been trying to think about how best to approach teaching Rust to those used to working with Java, in order to bring a group of developers up to speed with the language for a new project. Java was the language I learned at university, but it was a long time ago and I've been out of the Java world for a while. Not to date myself too much, but back when I was writing Java, if you wanted to pass a function as an argument, you had to declare a new interface or wrap a function in `Callable<T>`. Java has come along way since then. It's added many features from the ML lineage of languages and some functional programming influences. There are lambda's now, `Optional` types, etc.

There are a lot of great free resources out there to learn Rust, so don't take this post as in any way complete, just some thoughts on the topic.

I think the salient points to learn coming from Java to Rust basically boil down to a few broad categories:

- Differences in the memory model (lack of GC, etc)
- Ownership
- Algebraic Data Types
- Parametric Polymorphism (Generics) & Traits
- Lifetimes & Borrowing

I'm interested to hear from other folks also; if you have some thoughts on pedagogy feel free to shoot me an e-mail. You can find me on the Rust or Tokio discord chats sporadically also.

## No GC

Rust doesn't have a garbage collector. You have much more control over how you allocate and where your values live. By default, everything exists on the stack. So this,

```rust
let a = Thing { field: 1 };
```

Doesn't require any kind of dynamic memory allocation. The compiler can figure out the exact number of bytes that this is going to take, there's even a trait for it called `Sized`.

However, we can create heap values with `Box`, and we can even create something close to Java's objects, and gain back dynamic dispatch with traits.

```rust
trait Foo {}

struct A {}

impl Foo for A {}

fn main () {
    let a = Box::new(A {}); // the type here is Box<dyn Foo>
}
```

There is a cost to get this dynamic behaviour though. We pay by being explicit and with the actualy cost of heap allocation. There's some debate on the topic but I think it's fair to say it's idiomatic to avoid heap allocation if it is easily avoidable.

I found visual diagrams to be eminently helpful in getting all of this to sink in. The [Rust container cheat sheet](https://docs.google.com/presentation/d/1q-c7UAyrUlM-eZyTo1pd8SZ0qwA_wYxmPZVOQkoDmH4/edit#slide=id.p) is a great resource.

## Algebraic Data Types

Programming in Rust is a much more data-centric, type driven approach to programming. Rust has 2 main ways of declaring new types of values, the `enum` keyword and `struct`. `enum` is a sum type (a tagged union if you prefer). I don't think it's helpful to think of a Rust enum as analogous to `enum` in Java. It does express that a type can have different variants, but it's much more powerful. For example, you may have heard that Rust lacks `null` or `nil`. This is true, if you want to express the absence of a value in Rust, there's a type in the [stdlib](https://doc.rust-lang.org/std/option/enum.Option.html) for that.

```rust
enum Option<T> {
    None,
    Some(T),
}
```

In English, this is declaring a new type `Option` that takes a type parameter `T`. `T` is unrestricted or unbounded -- meaning the definition is valid for any type, and we're representing that with the variable `T`. `Option` has 2 possible variants, it's either `None` representing the lack of a value, or it is something and we have a value of `T`. In practice,

```rust
let a = Some("foo".to_string()); // declaring a value of type Option<String>
let b: Option<usize> = Some(1); // declaring a value of type Option<usize>, but with a type annotation
```

A note about type inference, Rust has "local" inference, and it's considered idiomatic to leave the annotation off if you can get away with it. There are some cases where it is needed, but we can get to that another time.

Rust includes a way to pattern match on `enum` variants with the `match` keyword. If you haven't used a language with robust pattern matching before, it's really a pleasure to use.

```rust
fn plus(a: Option<usize>) -> Option<usize> {
    match a {
        Some(v) => v + 1,
        None
    }
}
```

This is a simple case, it's pattern language much more powerful; you could write a whole article just about `match`. Check out the [cheats.rs](https://cheats.rs/#pattern-matching) listing of valid syntax for match. It replaces `if/else` expressions in a lot of cases.

Here we take an `Option` value, add something to it (if it is a `Some` variant) and return that value. This is a common pattern, and `Option` has funcions in the standard library to write this tersely.

```rust
fn plus(a: Option<usize>) -> Option<usize> {
    a.map(|v| v + 1)
}
```

`|| {}` is the syntax for closures. Closures in Rust don't require any heap allocation by default.

`struct` is the way to declare new records. This will probably be more familiar to anyone with a background in C-based languages.

```rust
struct A {
    field: usize
}
```
