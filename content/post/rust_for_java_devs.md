---
title: "Rust for Java Devs"
date: 2020-02-02T22:23:50-05:00
draft: true
---

In a change for this blog post I want to talk leave the more advanced topics and focus on perhaps one of the most important things one can do in the Rust community; namely, teaching new Rust developers. I've been trying to think about how best to approach teaching Rust to those used to working with Java, in order to bring a group of developers up to speed with the language for a new project. Java was the language I learned at university, but it was a long time ago and I've been out of the Java world for a while. Not to date myself too much, but back when I was writing Java, if you wanted to pass a function as an argument, you had to declare a new interface or wrap a function in `Callable<T>`. Java has come along way since then. It's added many features from the ML lineage of languages and even some functional programming influences. There are lambda's now, `Optional` types, etc. There are a lot of great free resources out there to learn Rust, so don't take this post as in any way complete, just some thoughts on the topic.

I think the salient points to learn coming from Java to Rust basically boil down to a few broad categories:

- Differences in the memory model (lack of GC, etc)
- Algebraic Data Types
- Ownership
- Lifetimes & Borrowing
- Parametric Polymorphism (Generics) & Traits

I'm interested to hear from other folks also; if you have some thoughts on pedagogy feel free to shoot me an e-mail. I'll cover the first 2 of these for now with a little more about traits & polymorphism at the end.

## No GC

Rust doesn't have a garbage collector. You have much more control over how you allocate and where your values live. By default, everything exists on the stack. So this,

```rust
let a = Thing { field: 1 };
```

Doesn't require any kind of dynamic memory allocation. The compiler can figure out the exact number of bytes that this is going to occupy in memory, there's even a trait for this called `Sized`. Not to jump ahead too much, but it should be noted by default all generic parameters are implicitly bounded by `Sized`, meaning if you write `fn foo<T>(t: T) -> T`, the it's implied that `T: Sized`. You have to specifically opt out of this constraint with `T: ?Sized`. Specifying that `T` _might_ be unsized. For more info, check out the [Unsized Types](https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait) chapter.

But I digress. We can create heap values with `Box`, and we can even create something close to Java's objects, and gain back dynamic dispatch with traits.

```rust
trait Foo {}

struct A {}

impl Foo for A {}

fn main () {
    let a = Box::new(A {}); // the type here is Box<dyn Foo>
}
```

This type is `Sized` also, it's just its size is that of a `Box`. I found visual diagrams to be eminently helpful in getting all of this to sink in. The [Rust container cheat sheet](https://docs.google.com/presentation/d/1q-c7UAyrUlM-eZyTo1pd8SZ0qwA_wYxmPZVOQkoDmH4/edit#slide=id.p) is a great resource.

This dynamic behaviour isn't free. We pay by being explicit and with the actual cost of heap allocation. There's some debate on the topic but I think it's fair to say it's idiomatic to avoid heap allocation if it is easily avoidable.

## Algebraic Data Types

### Enum

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

`enum` is yet even more powerful than this though, it has the ability to declare full recursive types. Watch:

```rust
enum List<T> {
    Nil,
    Cons((T, List<T>))
}
```

If you've ever used a FP lang this may look familiar. `List<T>` can either be `Nil` meaning it hit the end of the list, or a `Cons` tuple holding a value and the rest of the list. Think about how this will look in memory for a second, with all we've talked about Rust's memory model...

Huh?

It doesn't work.

```bash
error[E0072]: recursive type `List` has infinite size
 --> src/lib.rs:7:1
  |
7 | enum List<T> {
  | ^^^^^^^^^^^^ recursive type has infinite size
8 |     Nil,
9 |     Cons((T, List<T>))
  |          ------------ recursive without indirection
  |
  = help: insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to make `List` representable
```

That error message is pretty good. It's telling us we can't make a recursive type like this without adding in some indirection so that the compiler can figure out it's size. Without a reference or a pointer to the next element on the list, how will the compiler statically figure out how big the type is? Convince yourself this is true, thinking visually sometimes helps.

We can `fix` this by adding indirection:

```rust
enum List<T> {
    Nil,
    Cons((T, Box<List<T>>))
}
```

Now we have the worlds most terrible linked list. If you liked this example, consider reading [too many linked lists](http://cglab.ca/~abeinges/blah/too-many-lists/book/).

### Struct

`struct` is the way to declare new records. This will probably be more familiar to anyone with a background in C-based languages.

```rust
struct A {
    field: usize
}
```

You can also declare an 'anonymous' record with the tuple syntax.

```rust
let a: (usize, usize) = (1, 1); // again, type annotation not needed
```

Structs can and often to hold generic types as well.

```rust
struct Foo<T> {
    field: T
}
```

In my opinion, a good handle on Rust starts with a understanding of the basic data type definitions. `enum` and `struct` will be your bread and butter. Moving on, I don't know that I could explain ownership, borrowing & lifetimes better than it's already been explained elsewhere, but I think I can contribute something about the differing approach to encoding problems in Rust.

## Polymorphism

In Java polymorphism is usually means inheritance. That's not really true outside the OO world. Rust does not have OO inheritance. You declare data structures with `enum` or `struct` (or both) and define it's implementation with `impl` and/or `impl` various existing traits in order to extend your type with functionality. Let's take an example,

There are many different string types in Rust. There's `str`, `String` (and it's referenced siblings `&str`, `&String`), `OsStr`, `OsString`, (`&OsStr`, `&OsString`). So if you declare a function that you want to take a string, which type should you use?

`&str` is a good choice, you can pass a `&str` or `&String` to it and it will work because `String` implements the trait `Deref<Target=str>`. But we could do better than that with a polymorphic type.

```rust
fn foo<S: AsRef<str>>(s: S) {
    unimplemented!()
}
```

`AsRef` is a conversion trait in the [stdlib](https://doc.rust-lang.org/std/convert/trait.AsRef.html). It's defined as:

```rust
trait AsRef<T>
where
    T: ?Sized,
{
    fn as_ref(&self) -> &T;
}
```

It's a trait that takes a type parameter T, that _may_ be unsized, and defines a function which turns that type into a `&T`. If we call our original function `foo` with a `String`,

```rust
let a = String::from("stuff");
foo(a);
```

Why does this work? See if you can figure it out for yourself. If you need a [hint](https://doc.rust-lang.org/std/string/struct.String.html#implementations). When writing idiomatic Rust, it's crucial to get to know the built in [conversion traits](https://stevedonovan.github.io/rustifications/2018/09/08/common-rust-traits.html) available in the stdlib.

Very broadly, traits are used to encapsulate behaviours that a type can have. A type implements these behaviours in order to gain that set of functionality. In a lot of ways it works like an interface in Java (especially since Java 8 introduced default implementations in interfaces).
