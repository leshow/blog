---
title: "Rust for Java Devs"
date: 2020-02-02T22:23:50-05:00
draft: true
---

In a change for this blog I want to leave the more bleeding edge topics and focus on perhaps one of the most important things one can do in the Rust community: teaching new Rust developers. I've been thinking about how best to approach teaching Rust to those used to working with Java, in order to bring a group of developers up to speed with the language for a new project.

Java was the language I learned & abused in university, but it was a long time ago and I've been out of the Java world for a while. Not to date myself too much, but back when I was writing Java, if you wanted to pass a function as an argument, you had to declare a new interface or wrap a function in `Callable<T>`. Java has come along way since then. It's added features that had a clear influence from functional programming and the ML lineage of langs. We've got lambda's now, `Optional` types, etc. This article isn't going to be a comparison of Rust & Java, nor am I going to tell you that Rust is better for everything, I'm just hoping to introduce a few concepts that might be foreign to the average Java dev. There are a lot of great free resources out there to learn Rust, so don't take this post as in any way comprehensive.

I think the salient points to learn coming from Java to Rust basically boil down to a few broad categories:

- Differences in the memory model (lack of GC)
- Algebraic Data Types
- Ownership
- Lifetimes & Borrowing
- Parametric Polymorphism (Generics) & Traits

I'm interested to hear from other folks also; if you have some thoughts on pedagogy feel free to shoot me an e-mail. I'll cover the first 2 of these for now with a little more about traits & polymorphism at the end.

## Intro

Before starting any language it's good to check what problems it intends to solve, design decisions only make sense in the context of their stated goals. For Java, the goals of the language are:

    It must be simple, object-oriented, and familiar.
    It must be robust and secure.
    It must be architecture-neutral and portable.
    It must execute with high performance.
    It must be interpreted, threaded, and dynamic.

[source](<https://en.wikipedia.org/wiki/Java_(programming_language)#Principles>)

Rust's goals are:

    Performance - Rust is blazingly fast and memory-efficient: with no runtime or garbage collector, it can power performance-critical services, run on embedded devices, and easily integrate with other languages.

    Reliability - Rust’s rich type system and ownership model guarantee memory-safety and thread-safety — enable you to eliminate many classes of bugs at compile-time.

    Productivity - Rust has great documentation, a friendly compiler with useful error messages, and top-notch tooling — an integrated package manager and build tool, smart multi-editor support with auto-completion and type inspections, an auto-formatter, and more.

[source](https://www.rust-lang.org/)

There are some big differences here. Performance is first on Rust's list, it mentions "no runtime or garbage collector", and embedded devices. This is a systems language. In the second goal we see having a powerful type system is a stated goal, and in the last, a friendly compiler and ecosystem.

## Stack vs Heap

### The Stack

Rust doesn't have a garbage collector. You have much more control over how you allocate and where your values live. By default, everything exists on the stack. So this,

```rust
struct Thing { field: usize }

fn main() {
    let a = Thing { field: 1 };
    // .. do stuff with a
}
```

Doesn't require any kind of dynamic memory allocation. The compiler can figure out the exact number of bytes that this is going to occupy in memory, there's even a trait for this called `Sized`. The compiler can also figure out how long this value is valid for. It has a defined starting point when it was created, and when it goes out of scope (at the end of main) and it can be destroyed.

Contrast this with Java, where you create instances of an object with `new`. `new` causes heap allocation, and a reference will be stored in your variable and passed by value.

### The heap

We can make heap values in Rust too with `Box` (there are other smart pointers in the stdlib which also allocate, `Rc`, `Arc`, etc). You may ask why we would ever want to heap allocate if we can just keep putting things on the stack? The answer is varied, but one response has to do with the fact that not all types have a size that's statically available, so we can heap allocate to make the size known (we'll do an example of this). Other times, you may have a large bit of data, like a large struct, which would normally case a big copy when it gets moved around, by putting it behind a `Box` or other smart pointer, we can shrink the amount of data we move at the cost of the allocation.

```rust

fn main () {
    let a = Box::new(Thing { field: 1 });
}
```

Heap allocated values have a defined beginning and end too. There is a starting and ending scope for `a` when it will be created and destroyed. I find it helpful to think about what the value actually _looks_ like that's in `a`. In the first example it was a struct with a single `usize` in it, but here the value on the stack is a pointer to some memory on the heap.

Another use for `Box` is dynamic dispatch. With this we can get something like Java's objects,

```rust
trait Foo {}

struct Thing {}
impl Foo for Thing {}

struct OtherThing {}
impl Foo for OtherThing {}

fn main () {
    let a: Vec<Box<dyn Foo>> = vec![Box::new(OtherThing {}), Box::new(Thing { })]; // type optional
    // we have 2 different structs in the same list here
    // but we've erased the concrete type and now only know it as Foo
}
```

This dynamic behaviour isn't free. We pay by being explicit in our program, and with the actual cost of heap allocation. There's some debate on the topic but I think it's fair to say it's idiomatic to avoid heap allocation if it is easily avoidable.

I found visual diagrams to be eminently helpful in getting all of this to sink in. The [Rust container cheat sheet](https://docs.google.com/presentation/d/1q-c7UAyrUlM-eZyTo1pd8SZ0qwA_wYxmPZVOQkoDmH4/edit#slide=id.p) is a great resource.

## Algebraic Data Types

### Enum

Programming in Rust is a much more data-centric, type driven approach to programming. Rust has 2 main ways of declaring new types of values, the `enum` keyword and `struct`. `enum` is a sum type (a tagged union if you prefer). I don't think it's helpful to think of a Rust `enum` as analogous to `enum` in Java, but the common theme here is that both express that a type can have different variants. The Rust version is capable of expressing a great deal more. For example, you may have heard that Rust lacks `null` or `nil`. This is true, if you want to express the absence of a value in Rust, there's a type in the [stdlib](https://doc.rust-lang.org/std/option/enum.Option.html) for that.

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

A note about type inference, Rust has "local" inference, and it's considered idiomatic to leave the annotation off if you can get away with it. There are some cases where it is needed, but that's outside the scope of todays post.

Rust includes a way to pattern match on `enum` variants with the `match` keyword. If you haven't used a language with robust pattern matching before, it's really a pleasure to use.

```rust
fn plus(a: Option<usize>) -> Option<usize> {
    match a {
        Some(v) => v + 1,
        None
    }
}
```

This is a simple case, it's pattern language is much more powerful; you could write a whole article just about `match`. Check out the [cheats.rs](https://cheats.rs/#pattern-matching) listing of valid syntax for `match`. It replaces `if/else` expressions in a lot of cases.

Here we take an `Option` value, add something to it (if it is a `Some` variant) and return that value. This is a common pattern, and `Option` has functions in the standard library to write this tersely.

```rust
fn plus(a: Option<usize>) -> Option<usize> {
    a.map(|v| v + 1)
}
```

`|| {}` is the syntax for a closure. Closures in Rust don't require any heap allocation, this is in line with the goals of the language which aims to provides us with a "rich type system" at no cost to performance.

`enum` is even more powerful than this though, it has the ability to declare fully recursive types.

```rust
enum List<T> {
    Nil,
    Cons((T, List<T>))
}
```

This probably looks pretty foreign. `List<T>` can either be `Nil` meaning it hit the end of the list, or a `Cons` tuple holding a value and the rest of the list. Think about how this will look in memory if you were to create a value for it.

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

The error messages from the compiler are really top notch. There's a reason and often `help` has a fix. It's telling us we can't make a recursive type like this without adding in some indirection so that the compiler can figure out it's size. Remember, if everything is on the stack by default, and stack values need to have a statically known size, then how can we have an n-sized linked list? Without a reference or a pointer to the next element on the list, how will the compiler statically figure out how much memory to use? Convince yourself this is true, thinking visually sometimes helps.

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

Structs can and often do hold generic types as well.

```rust
struct Foo<T> {
    field: T
}
```

### impls

With both `enum` and `struct` we can make an implementation for the data type we have defined. I find it best to think of an `impl` as a set of transformations available to your type. This is about as close to the notion of OO as we're going to get in Rust. You can have a `struct` with an `impl`, and if you squint hard enough it _almost_ looks like an object:

```rust
struct Thangs {
    list: Vec<Thang>
}

struct Thang {}

impl Thangs {
    // *note* there is nothing special about `new` here, it's just convention
    fn new() -> Self {
        Self {
            list: vec![]
        }
    }

    fn add_thang(&mut self, thang: Thang) {
        self.list.push(thang);
    }
}

fn main() {
    // *note* the &mut self in add_thangs requires us to declare mut below
    let mut thangs = Thangs::new();
    thangs.add_thang(Thang {});
}
```

Contrasting with Java, the `value.method()` syntax actually is actually just sugar over a "universal function call syntax". We could call the method by passing `&mut self` in ourselves:

```rust
let mut thangs = Thangs::new();
Thangs::add_thang(&mut thangs, Thang {});
```

In my opinion, a good handle on Rust starts with a understanding of the basic data type definitions. `enum` and `struct` will be your bread and butter.

## Traits

The traits & generics (you may see it call "parametric polymorphism" sometimes) implementation in Rust is robust. You may have heard it compared to operator overloading, which Java lacks. I think this is a fair introduction to it's feature set. In Java, overloading is shunned. Not so in Rust, traits are provided for you to implement and conform to their specification, gaining access to built-in syntax and interoperability. Consider that you can plug-in to the language's syntax with traits; that's how the whole ecosystem works. There's the `Future` trait for await-able computations, there's `Iterator` and `IntoIterator` to use `for..in`, `Index` for `[]`, not to mention `Add`, `Sub`, `Mul`, etc for arithmetic operations. As a minimal example, let's make a type work with [`Add`](https://doc.rust-lang.org/std/ops/trait.Add.html)

Here is our `Add` definition from std:

```rust
pub trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Self::Output;
}
```

```rust
use std::ops::Add;

#[derive(Debug)]
struct Content<T> { // 1
    val: T,
}

impl<T> Add for Content<T> // 2
where
    T: Add, // 3
{
    type Output = Content<<T as Add>::Output>; // 4
    fn add(self, rhs: Content<T>) -> Self::Output {
        Content {
            val: self.val + rhs.val,
        }
    }
}

fn main() {
    let a = Content { val: 2 };
    let b = Content { val: 5 };
    println!("{:?}", a + b); // Content { val: 7 }
}
```

We're declaring a new type `Content` that's valid for any `T` (1). In the implementation for `Add`, we say that `Content` has an `Add` implementation (2) so long as the thing that's in `Content` _also_ has an `Add` impl (3). After that, we specify the `Output` associated type is going to be `Content` of `Output` of `T` when it impls `Add` (4). Don't worry if that all doesn't make sense at first, once you write a few implementations it will start to click.

It should be noted by default all generic parameters are implicitly bounded by `Sized`, meaning if you write `fn foo<T>(t: T) -> T`, it's implied that `T: Sized`. You have to opt out of this constraint with `T: ?Sized`. Specifying that `T` _might_ be unsized. For more info, check out the [Unsized Types](https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait) chapter.

## Polymorphism

In Java polymorphism usually means inheritance. That's not really true outside the OO world, certainly not in Rust, which lacks the feature altogether. You declare data structures with `enum` or `struct` (or both) and define it's implementation with `impl` and/or `impl` various existing traits in order to extend your type with functionality. Let's take an example,

There are many different string types in Rust. There's `str`, `String`, `OsStr`, `OsString`, `CString` and `CStr` (did I miss any?). Practically, we usually only use `str` and `String`, the other's are special purpose. But if we declare a function that we want to take a string, which type should we use?

`&str` is a good choice, you can pass a `&str` or even `&String` and it will work because `String` implements the trait `Deref<Target=str>` (the `Target=` syntax means that `Target` is an "associated type"). But we could be more generic with a polymorphic type.

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

Why does this work? See if you can figure it out for yourself. If you need a [hint](https://doc.rust-lang.org/std/string/struct.String.html#implementations). When writing idiomatic Rust, it's crucial to get to know the built in [conversion traits](https://stevedonovan.github.io/rustifications/2018/09/08/common-rust-traits.html) available in the stdlib. Very broadly, traits are used to encapsulate behaviours that a type can have. A type implements these behaviours in order to gain that set of functionality. In a lot of ways it works like an interface in Java (especially since Java 8 introduced default implementations in interfaces).

### A note on generics in Java v Rust

When Java was first released, they didn't include an implementation of generics. The feature was highly sought after because it allowed greater type safety and could remove some explicit casts. Java's bytecode has no concept of a generic type parameter though, after the compiler confirms that all the generic bounds are satisfied (specified in Java with `<T extends Class>`), Java does something called [type erasure](https://docs.oracle.com/javase/tutorial/java/generics/erasure.html). The basics of type erasure are that all type params are replaced with `Object` and are therefore unbounded in the final bytecode. I'm not going to critique Java's choice of implementation, but suffice to say there is one specific drawback I want to mention: indirection. Due to type erasure, generic arguments are passed as pointers to a vtable because we lost some information at compile time on the concrete type of the argument.

Rust's generics implementation doesn't use type erasure. In the `foo` method above, for each caller of `foo` that is using a different concrete type, a brand new version of `foo` will be generated and compiled. That means if `foo` get's called with 4 different implementors of `AsRef<str>` we could potentially get 4 different versions of the function in our final running code. This process is referred to as 'monomorphization'. The main advantage of this method is that it's fast and Rust's wants to provide "zero cost abstractions". All generic calls are statically dispatched, we don't have to heap allocate a new `Object` or pass around a vtable. It's really a joy to be able to create robust abstractions with traits and know the code generated will be no slower than if you hadn't use the abstraction. It should be mentioned that a disadvantage of this route is final code size and compilation time. Depending just how much polymorphism we use and just how many different variations there are of our functions, that's more code that has to go through LLVM lengthening compile times and increasing volume of actual code produced.

## Conclusion

There's such a breadth of topics that I could have possibly covered in this article, I really didn't know where to start. I hope I've at least illuminated something for the beginners out there. Feel free to contact me if you've got some ideas for new things you'd like to see explored. If you are just starting out with Rust and feeling a little overwhelmed, don't worry! All of these topics can feel new and different, the only cure is reading and writing lots of Rust. Stick with it!

Until next time.
