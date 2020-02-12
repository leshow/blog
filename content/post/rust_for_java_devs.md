---
title: "Rust for Java Devs"
date: 2020-02-02T22:23:50-05:00
draft: true
---

In a change for this blog I want to leave the more bleeding edge topics and focus on perhaps one of the most important things one can do in the Rust community: teaching new Rust developers. I've been thinking about how best to approach teaching Rust to those used to working with Java, in order to bring a group of developers up to speed with the language for a new project.

Java was the language I learned & abused in university, but it was a long time ago and I've been out of the Java world for a while. When I last wrote Java, if you wanted to pass a function as an argument, you had to declare a new interface or wrap a function in `Callable<T>`. Java has come along way since then. It's added features that have a clear influence from functional programming and the ML lineage of langs. I'm talking about lambda's, `Optional` types, etc. This article isn't going to tell you to write everything in Rust, or that you need to throw out all your Java code. Java is a great language with valid use cases. I want to explore some comparisons between Java and Rust for the budding Rust programmer.

## Motivation

First, a word on the goals of each language. Java was created to solve a set of problems. It was meant to be ["simple, object-oriented and modular"](<https://en.wikipedia.org/wiki/Java_(programming_language)#Principles>), it was intended to run in a virtual machine and be portable to different architectures, and it was meant to have high performance. Rust's goals are to be ["blazingly fast and memory efficient"](https://www.rust-lang.org/), have a "rich type system ... memory-safety and thread-safety", and be productive with good error messaging and a integrated package manager.

There are some differences here. Performance is first on Rust's list, it mentions not having a runtime while also being memory safe, with a powerful type system. These are the areas that Rust really shines; code that is performant and safe. Servers that need to handle many thousands of requests per second, applications that need to be fast and run with a small memory footprint, or maybe an OS or code running on an embedded device. These things can be done in other languages, but this is the domain that Rust was built for. Rust does this while also prevent things like buffer overflows, dangling or null pointer errors. That's not to say Rust isn't being used in other places (I'm looking at you wasm frontends).

From a pedagogical standpoint, I think the salient points to learn coming from Java to Rust basically boil down to a few broad categories:

- Algebraic Data Types
- Differences in the memory model (lack of GC)
- Ownership
- Lifetimes & Borrowing
- Parametric Polymorphism (Generics) & Traits

I'm interested to hear from other folks also; if you have some thoughts on pedagogy feel free to shoot me an e-mail. I won't cover all of these here, but we'll touch on some of the important parts.

## Algebraic Data Types

### Enum

Programming in Rust is a much more data-centric, type driven approach to programming. Rust has 2 main ways of declaring new types of values, the `enum` keyword and `struct`. `enum` is a sum type (a tagged union if you prefer). It's quite different `enum` in Java, but if we're going to make the comparison, think of it as like Java's `enum` but capable of expressing a great deal more. The common theme with `enum` in either language is is that both express a type can have different variants. For example, you may have heard that Rust lacks `null` or `nil`. This is true, if you want to express the absence of a value in Rust, there's a type in the [stdlib](https://doc.rust-lang.org/std/option/enum.Option.html) for that.

```rust
enum Option<T> {
    None,
    Some(T),
}
```

In English, this is declaring a new type `Option` that takes a type parameter `T`. `T` is unrestricted or unbounded -- meaning the definition is valid for any type, and we're representing that with the variable `T`. `Option` has 2 possible variants, it's either the `None` constructor (or 'data constructor') representing the lack of a value, or the `Some` constructor and a value of `T`. To declare values,

```rust
let a = Some("foo".to_string()); // declaring a value of type Option<String>
let b: Option<usize> = Some(1); // declaring a value of type Option<usize>, but with a type annotation
```

A note about type annotations: it's required at function boundaries but locally types are inferred. It's considered idiomatic to leave the annotation off if you can get away with it. There are some cases where it is needed, but that's outside the scope of todays post.

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

### Impl

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

In my opinion, a good handle on Rust starts with an understanding of the basic data type definitions. `enum` and `struct` will be your bread and butter. In Java, something like `struct` and `impl` are stuck together in objects, where your data and methods cohabitate, this couples code together (I suppose intentionally). Before you think "well why can't Rust just add objects", Java is also getting a struct-like feature. Coming in Java 14, "Records" will be added to the language. So it may actually be the case that next generation of Java code will end up looking more rust-ic than the inverse (I'm in no way claiming Rust was the first to do sum & product types). I've even seen proposals in Java that have something like sum types, so go ahead, embrace algebraic data types!

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

Doesn't require any kind of dynamic memory allocation. The compiler can figure out the exact number of bytes that this is going to occupy in memory, there's even a trait for this called `Sized`. The compiler can also figure out how long this value is valid for. It has a defined starting point when it was created, and when it goes out of scope (at the end of main) it can be destroyed.

Contrast this with Java, where you create instances of an object with `new`, which causes heap allocation and an implicit reference be stored in your variable, which is passed around by-value.

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

Worry not about the `trait`, we'll visit this also. The important thing to gather here is that this dynamic behaviour isn't free. We pay by being explicit in our program, and with the actual cost of heap allocation. There's some debate in the Rust community, but I think it's fair to say it's idiomatic to avoid heap allocation if it is easily avoidable.

I found visual diagrams to be eminently helpful in getting all of this to sink in. The [Rust container cheat sheet](https://docs.google.com/presentation/d/1q-c7UAyrUlM-eZyTo1pd8SZ0qwA_wYxmPZVOQkoDmH4/edit#slide=id.p) is a great resource.

Lets take a look at a another type defintion,

```rust
enum List<T> {
    Nil,
    Cons((T, List<T>))
}
```

This might look pretty foreign. `List<T>` can either be `Nil` meaning it hit the end of the list, or a `Cons` tuple holding a value and the rest of the list. Think about how this will look in memory if you were to create a value for it.

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

The error messages from the compiler are really top notch. There's a reason given and often `help` has a fix. It's telling us we can't make a recursive type like this without adding in some indirection so that the compiler can figure out it's size. Remember, if everything is on the stack by default, and stack values need to have a statically known size, then how can we have an n-sized linked list? Without a reference or a pointer to the next element on the list, how will the compiler statically figure out how much memory to use? Convince yourself this is true, thinking visually sometimes helps.

We can `fix` this by adding indirection,

```rust
enum List<T> {
    Nil,
    Cons((T, Box<List<T>>))
}
```

Now we have the worlds most terrible linked list. If you liked this example, consider reading [too many linked lists](http://cglab.ca/~abeinges/blah/too-many-lists/book/). You may find smart pointer types in definitions from time to time

## Traits

### Motivating example

To illustrate some differences in encoding problems in Java v Rust, let's look at another (admittedly toy) problem:

```rust
enum Shape {
    Circle { radius: f32 },
    Rectangle { width: f32, height: f32 },
}
```

We want to get the `area` for this type, so maybe we make a function:

```rust
impl Shape {
    pub fn area(self: Area) -> f32 {
        match area {
            Circle { radius } => std::f32::consts::PI * r.powi(2),
            Rectangle { width, height } => width * height,
        }
    }
}
```

Encapsulation & visibility in Java has a lot of forms at the class level. In Rust, functions and types are either `pub` (for public), or not (local to the module). Visibility to other members is controlled by the module system, I recommend reading the [Rust reference](https://doc.rust-lang.org/reference/visibility-and-privacy.html).

Back to our example. In Java, we may make a `Shape` parent class or interface, and have a `Circle` and `Rectangle` class, each implementing the `area` method. If we think about the differences between the Rust and Java implementation, a few things become clear:

- If we have to add another `Shape`:
  - in Java we just need to declare another class and implement `Shape`
  - in Rust we have to modify the original `Shape` definition _and_ everywhere it's used (`match` will refuse to compile unless it handles all the possible variants)
- If we add a new function for `Shape`:
  - we have to modify the original `Shape` 'contract' in Java, meaning we had to add a new function to the interface, and everyone who implements this interface must be changed
  - in Rust we can just add a new `impl`

This has been described before as the 'expression problem'. It illustrates some central differences between approaches to languages. This isn't meant to detract from using `enum` in Rust or using interfaces/classes in Java. There are many places where you _want_ Rust to do exhaustiveness analysis with `match` and where a sum type just makes the most sense. But it begs the question, "can we come up with a system where we don't have to modify the original definition in either case?"

I think traits provide a pretty nice solution,

```rust
trait Area {
    fn area(self) -> f32; // we could even return a generic or associated type here
}

struct Rectangle {
    width: f32,
    height: f32
}

struct Circle {
    radius: f32
}

impl Area for Rectangle {
    fn area(self) -> f32 {
        self.width * self.height
    }
}

impl Area for Circle {
    fn area(self) -> f32 {
        std::f32::consts::PI * self.radius.powi(2)
    }
}
```

Now, if we need to add a new function to `Circle` or `Rectangle`, say `perimeter`, we can do it without modifying the original type definition.

```rust
trait Perimeter {
    fn perimeter(self) -> f32;
}

impl Perimeter for Rectangle {
    fn perimeter(self) -> f32 {
        2. * (self.width + self.height)
    }
}
// etc
```

Additionally, we can write functions that take any type which has a perimeter, or perimeter _and_ area,

```rust
fn do_something<T: Perimeter + Area>(shape: T) { // only accept types who have a Perimeter and Area impl
    unimplemented!() // hot tip: this macro is invaluable. it satisfies the type checker without providing an implementation
}
```

### Traits & Generics

The traits (called "parametric polymorphism" sometimes) implementation in Rust is expressive. You may have heard it compared to operator overloading, which Java lacks. I think this is a fair introduction to it's feature set. In Java, overloading is shunned. Not so in Rust, traits are provided for you to implement and conform to their specification, gaining access to built-in syntax and interoperability. Consider that you can plug-in to the language's syntax with traits; that's how the whole ecosystem works. There's the `Future` trait for await-able computations, there's `Iterator` and `IntoIterator` to use `for..in`, `Index` for `[]`, not to mention `Add`, `Sub`, `Mul`, etc for arithmetic operations. As a minimal example, let's make a type work with [`Add`](https://doc.rust-lang.org/std/ops/trait.Add.html)

Here is our `Add` definition from std,

```rust
pub trait Add<Rhs = Self> { // 1
    type Output; // 2
    fn add(self, rhs: Rhs) -> Self::Output; // 3
}
```

Std declares a trait `Add` with a type parameter called `Rhs` that defaults to `Self`-- the implementor of the trait (1). It has an "associated type" called `Output` (2) and defines a method called `add` that takes `self` by value (takes ownership of `self`) and a parameter `rhs` of type `Rhs` (the type param passed in), and returns the type associated with `Output` (3).

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

We're declaring a new type `Content` that's valid for any `T` (1). In the implementation for `Add`, we say that `Content` has an `Add` implementation (2) so long as the thing that's in `Content` _also_ has an `Add` impl (3). After that, we specify the `Output` associated type is going to be `Content` of `Output` of `T` when it impls `Add` (4). Don't worry if that all doesn't make sense at first, once you write a few implementations it will start to click. I think it's kind of cool that most of this program is actually about the type system. We only have a few lines that actually "do" anything, this should give you a feel for what programming in Rust is like. You are predominantly designing at the type level, then convincing Rust your program is well-formed, and it makes a pretty good pair programmer at pointing out your errors.

It should be noted by default all generic parameters are implicitly bounded by `Sized`, meaning if you write `fn foo<T>(t: T) -> T`, it's implied that `T: Sized`. You have to opt out of this constraint with `T: ?Sized`. Specifying that `T` _might_ be unsized. For more info, check out the [Unsized Types](https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait) chapter.

## Polymorphism

In Java polymorphism usually means inheritance. That's not really true outside the OO world, certainly not in Rust, which lacks Java's version of the feature altogether. To use the PLT terminology, Java's brand of polymorphism is subtype polymorphism, whereas Rust is parametric and ad-hoc polymorphism. Don't feel bad if those words don't mean much to you. The important things to know are that you declare data structures with `enum` or `struct` (or both) and define it's implementation with `impl` and/or `impl` various traits in order to extend your type with functionality, and you can bound functions by allowing only certain implementors to call. Let's take an example,

There are many different string types in Rust. There's `str`, `String`, `OsStr`, `OsString`, `CString` and `CStr` (did I miss any?). Practically, the common ones are `str` and `String`, the other's are special purpose. If we declare a function that we want to take a string, which type should we use?

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

Why does this work? See if you can figure it out for yourself (if you need a [hint](https://doc.rust-lang.org/std/string/struct.String.html#implementations)). When writing idiomatic Rust, it's crucial to get to know the built in [conversion traits](https://stevedonovan.github.io/rustifications/2018/09/08/common-rust-traits.html) available in the stdlib. Very broadly, traits are used to encapsulate behaviours that a type can have. A type implements these behaviours in order to gain that set of functionality. In a lot of ways it works like an interface in Java (especially since Java 8 introduced default implementations in interfaces). Java naturally lends itself to a style that's very inheritance-centric. However, it's popular in the OO world to repeat the mantra "composition over inheritance". In Rust, you don't have an option, it's composition all the way.

## Conclusion

There's such a breadth of topics that I could have possibly covered in this article, I really didn't know where to start. I hope I've at least illuminated something for the beginners out there. Feel free to contact me if you've got some ideas for new things you'd like to see explored. If you are just starting out with Rust and feeling a little overwhelmed, don't worry! All of these topics can feel new and different, the only cure is reading and writing lots of Rust. Stick with it!

Until next time.

#### Note: on generics in Java v Rust

When Java was first released, they didn't include an implementation of generics. The feature was highly sought after because it allowed greater type safety and could remove some explicit casts. Java's bytecode has no concept of a generic type parameter though, and it was important to maintain backwards compatibility. So, after the Java compiler confirms that all the generic bounds are satisfied (specified in Java similarly with `<T>` or `<T extends Class>`), Java does something called [type erasure](https://docs.oracle.com/javase/tutorial/java/generics/erasure.html). The basics of type erasure are that all type params are replaced with `Object` and are therefore unbounded in the final bytecode. I'm not going to critique Java's choice of implementation, but suffice to say there is one specific drawback I want to mention: indirection. Due to type erasure, generic arguments are passed as pointers to a vtable because we lost some information at compile time on the concrete type of the argument.

Rust's generics implementation doesn't use type erasure. In the `foo` method above, for each caller of `foo` that is using a different concrete type, a brand new version of `foo` will be generated and compiled. That means if `foo` get's called with 4 different implementors of `AsRef<str>` we could potentially get 4 different versions of the function in our final running code. This process is referred to as 'monomorphization'. The main advantage of this method is that it's fast and Rust's wants to provide "zero cost abstractions". All generic calls are statically dispatched, we don't have to heap allocate a new `Object` or pass around a vtable. It's really a joy to be able to create robust abstractions with traits and know the code generated will be no slower than if you hadn't use the abstraction. It should be mentioned that a disadvantage of this route is final code size and compilation time. Depending just how much polymorphism we use and just how many different variations there are of our functions, that's more code that has to go through LLVM lengthening compile times and increasing volume of actual code produced.
