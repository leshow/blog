---
title: "Cheating Higher Ranks with Traits"
date: 2020-03-06T23:37:54-05:00
draft: true
---

I'll begin by describing a recent problem I had when trying to abstract a bit of code. I had an enum that described a set of possible branches, for each branch there was an associated type to run through a serialize function.

```rust
enum Var {
    One,
    Two,
}

#[derive(Serialize)]
struct Foo;

#[derive(Serialize)]
struct Bar;


fn write<W>(var: Var, mut writer: W) -> serde_json::Result<()>
where
    W: Write,
{
    match var {
        Var::One => serde_json::to_writer(&mut writer, &Foo),
        Var::Two => serde_json::to_writer(&mut writer, &Bar),
    }
}

```

Life was good. Then I realized that I actually needed to format these types in two different ways, one with `to_writer` and the other with `to_writer_pretty`. I could make a `write` and `write_pretty` function, but that felt dirty. The only thing that would be different in each implementation is which function from `serde_json` it called.

No problem, I thought, I'll parameterize the formatting function and pass it as a closure.

```rust
fn write<W, T, F>(var: Var, mut writer: W, f: F) -> serde_json::Result<()>
where
    W: Write,
    T: Serialize + ?Sized,
    F: Fn(&mut W, &T) -> serde_json::Result<()>,
{
    match var {
        Var::One => f(&mut writer, &Foo),
        Var::Two => f(&mut writer, &Bar),
    }
}
```

But this was folly, I'm declaring `write` to be valid for any `T`, then selecting some concrete `T` in the implementation. Indeed, this will generate an error.

```bash
error[E0308]: mismatched types
  --> src/lib.rs:23:36
   |
16 | fn write<W, T, F>(var: Var, mut writer: W, f: F) -> serde_json::Result<()>
   |             - this type parameter
...
23 |         Var::One => f(&mut writer, &Foo),
   |                                    ^^^^ expected type parameter `T`, found struct `Foo`
   |
   = note: expected reference `&T`
              found reference `&Foo`
   = help: type parameters must be constrained to match other types
   = note: for more information, visit https://doc.rust-lang.org/book/ch10-02-traits.html#traits-as-parameters

error[E0308]: mismatched types
  --> src/lib.rs:24:36
   |
16 | fn write<W, T, F>(var: Var, mut writer: W, f: F) -> serde_json::Result<()>
   |             - this type parameter
...
24 |         Var::Two => f(&mut writer, &Bar),
   |                                    ^^^^ expected type parameter `T`, found struct `Bar`
   |
   = note: expected reference `&T`
              found reference `&Bar`
   = help: type parameters must be constrained to match other types
   = note: for more information, visit https://doc.rust-lang.org/book/ch10-02-traits.html#traits-as-parameters
```

This is where higher ranks could come in. If I could declare `F: for<T> Fn(&mut W, &T) -> serde_json::Result<()>`. Basically, declaring that `T` is parameterized over the function `F` and not over `write`, then this would be valid. But such things are not allowed in Rust today.

How then to solve this problem?

## Type Erasure

I could 'erase' the type from the signature with a trait object,

```rust
fn write<W, T, F>(var: Var, mut writer: W, f: F) -> serde_json::Result<()>
where
    W: Write,
    T: Serialize + ?Sized,
    F: Fn(&mut W, &dyn Serialize) -> serde_json::Result<()>,
{
    match var {
        Var::One => f(&mut writer, &Foo),
        Var::Two => f(&mut writer, &Bar),
    }
}
```

The problem with this is that `Serialize` is not [object safe](https://doc.rust-lang.org/book/ch17-02-trait-objects.html#object-safety-is-required-for-trait-objects). If a trait has generic methods like `Serialize` does, it can't be turned into a trait object. I encourage you to read the link, it's enlightening.

There are crates like `erased_serde` that will allow one to make a `Box<dyn Serialize>`.

## Another Enum

I could make another enum, representing the different parameters that could be passed to the writer. This feels wrong, since it's what I'm trying to get away from in the first place.

```rust
enum Foobar<'a> {
    Foo(&'a Foo),
    Bar(&'a Bar)
}

fn write<W, T, F>(var: Var, mut writer: W, f: F) -> serde_json::Result<()>
where
    W: Write,
    T: Serialize + ?Sized,
    F: Fn(&mut W, FooBar) -> serde_json::Result<()>,
{
    match var {
        Var::One => f(&mut writer, Foo(&Foo)),
        Var::Two => f(&mut writer, Bar(&Bar)),
    }
}
```

But then my closure has to handle the multiple variants also,

```rust
write(var, writer, |writer, foobar| {
    match foobar {
        Foo(foo) => serde_json::to_writer_pretty(writer, foo),
        Bar(bar) => serde_json::to_writer_pretty(writer, bar),
    }
})
```

This isn't ideal either, I want to do the same thing for each type. Is there an easier way? I don't want to add a dependency, and I'm at the point now where this seems like an awful lot of work to try to abstract this bit of code. Maybe I should just copy-paste, make the modifications and be done with it?

## Cheating Rank-2

As usual in Rust-- traits to the rescue. It's possible to get around this by creating a trait with the behaviour I'm trying to abstract,

```rust
trait Format {
    fn format<W, T>(&self, writer: W, val: &T) -> serde_json::Result<()>
    where
        W: Write,
        T: Serialize + ?Sized;
}

struct Ugly;
struct Pretty;

impl Format for Ugly {
    fn format<W, T>(&self, writer: W, val: &T) -> serde_json::Result<()>
    where
        W: Write,
        T: Serialize + ?Sized,
    {
        serde_json::to_writer(writer, val)
    }
}

impl Format for Pretty {
    fn format<W, T>(&self, writer: W, val: &T) -> serde_json::Result<()>
    where
        W: Write,
        T: Serialize + ?Sized,
    {
        serde_json::to_writer_pretty(writer, val)
    }
}
```

Now, I can make `write` parameterized by this trait,

```rust
fn write<W, T, F>(var: Var, mut writer: W, format: F) -> serde_json::Result<()>
where
    W: Write,
    T: Serialize + ?Sized,
    F: Format,
{
    match var {
        Var::One => format.format(&mut writer, &Foo),
        Var::Two => format.format(&mut writer, &Bar),
    }
}
```

Elsewhere, I can simple call the `write` function with,

```rust
write(var, writer, Ugly)?;
```

Because `<T>` is bounded over the `format` function and not the trait itself, it's effectively the same thing as a rank-2 type (`for<T> Fn(T)`). I think this shows something interesting about type systems with traits or typeclasses. It's sometimes valuable to create new types even if they hold no data, even if their only purpose is to abstract a bit of behaviour and allow you to parameterize it.

If you don't require a trait definition to have object safety (`Format` won't be object safe), traits offer a convenient way to get around higher ranked types. Stick the additional type parameter in a trait method!
