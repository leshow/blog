---
title: "Serialization: Haskell & Rust"
date: 2019-03-20T22:52:17-04:00
draft: true
---

I recently published my first Haskell [package](https://hackage.haskell.org/package/i3ipc-0.1.0.0). While I'm almost sure I have no users yet, it was a great experience. I wrote a slim wrapper around i3's IPC, basically doing the the grunt work of writing all the necessary records for serialization, and a couple hundred lines of functions (mostly in IO) to interact with the common use cases (subscribing to events, retrieving data).

It probably wasn't worth the aggravation; my estimation is the cross section of Haskell users that also use tiling window managers would show a strong overlap with XMonad, not i3, but here we are. At least it gave an interest opportunity for comparison with a library in my other favorite language: Rust.

While there was no existing i3 IPC lib for Haskell, Rust does have a prime candidate: [i3ipc-rs](https://github.com/tmerr/i3ipc-rs/). It's a great library, but my guess is it was written a while ago because looking through it's source, most of the serde types could now be auto-derived instead of handwritten.

I'm working on a version of `i3ipc-rs` that uses tokio (essentially a copy of the Haskell library but in Rust/tokio), to compliment a fork of of another library that I converted to async IO. As a result, I've re-written all of the i3 IPC types a second time in Rust using serde, this library I plan to release standalone (as part of a cargo workspace) as soon as the accompanying project is finished, hopefully no one will ever have to write the types a third.

Enough exposition, let's get to the comparisons shall we?

## Field Names

### Haskell

I'm not sure if aeson was one of the first libraries to really popularize the whole auto-derived instances thing (I mean for programming in general, not just Haskell), but it certainly _feels_ like it. IMO, serde takes a lot of cues from aeson and improves on them.

In aeson, you can derive an instance with the help of a few language extensions (taken from `i3ipc`):

```haskell
{-# LANGUAGE DeriveGeneric, BangPatterns #-}

import GHC.Generics

data WorkspaceEvent = WorkspaceEvent {
    wrk_change :: !WorkspaceChange
    , wrk_current :: !(Maybe Node)
    , wrk_old :: !(Maybe Node)
} deriving (Eq, Generic, Show)

```

A couple things to note here. Firstly, the bang (!) patterns annotate the field as strict rather than lazy. And second, records in Haskell are a bit ornery on account of field names polluting the global namespace. As a result, I prefixed all of the field names with a keyword and underscore. The issue now, for the purposes of derivation, is that my field names no longer match the JSON keys in the structure.

Aeson has a solution for that:

```haskell
instance ToJSON WorkspaceEvent where
    toEncoding = genericToEncoding defaultOptions { fieldLabelModifier = drop 4 }

instance FromJSON WorkspaceEvent where
    parseJSON = genericParseJSON defaultOptions { fieldLabelModifier = drop 4 }
```

With this, when serializing to/from JSON, aeson will drop the first 4 characters and the names will match.

### Rust

Rust has an advantage here (IMO) that struct fields are locally namespaced. That gets around the antecedent keyword above. In Rust the derive mechanism is a special kind of macro called a proc macro. It looks like this:

```rust
#[derive(Deserialize, Serialize, PartialEq, Debug, Clone)]
pub struct WorkspaceData {
    pub change: WorkspaceChange,
    pub current: Option<reply::Node>,
    pub old: Option<reply::Node>,
}
```

`Serialize` and `Deserialize` come from serde, the other traits I'm deriving are part of Rust's std library and are among the 6 or 7 derivable traits for a struct. In this regard Rust is very similar to Haskell.

You may have noticed I didn't need to make any modifications to get the names on my struct to match the actual JSON, but if I did, serde has me covered there too with more proc macros:

```rust
#[derive(Deserialize, Serialize, PartialEq, Clone, Debug)]
pub struct Node {
    pub id: i32,
    pub name: Option<String>,
    #[serde(rename = "type")]
    pub node_type: NodeType,
    pub window_properties: Option<HashMap<WindowProperty, Option<String>>>,
    // ...removed
}
```

Note the `rename`. Serde has a bunch of these baked in, like `rename_all="snake_case"` or `lowercase`. Read about them [here](https://serde.rs/variant-attrs.html)

## Performance

## Conclusion

Haskell arguably has more flexibility here as `fieldLabelModifier` can be any function, but it seems that flexibility comes at a price. Briefly looking through the code for `serde_derive` it looks like serde will do the transformation at compile time (by virtue of `rename_by_rules` occurring inside `from_ast` [here](https://github.com/serde-rs/serde/blob/fa854a21083b06f509c537190550555e5473644f/serde_derive/src/internals/attr.rs#L1159)). I can't say definitively aeson won't do that with `fieldLabelModifier` but my guess is this is a run-time transformation, I'd happily like to be proven wrong though.

In a lot of ways the libraries function very similarly, and really, comparing the ergonomics of a library across languages is foolish. I did write less concrete instances in serde though.

--

`i3-tracker` is all synchronous, as is `i3ipc`, which irked me. The idea of spawning a new thread to do a timeout and potentially having those pile up unbounded-- while likely never to account for anything more than a momentary stutter (if that) screamed out to me "there has to be a better way". So, I did what any individual immersed in Rust of the times would do: I attempted to re-write it in tokio.

Tokio is super cool, it's also has a rep for being not very approachable. I'm not a systems developer, I write code for the web, so it has been a challenge. Still, I managed to fork `i3-tracker` and get everything running in a single thread without to much hassle. Then I went proudly look at my achievement in htop only to find 2 threads running:

- One for tokio that had all my futures + timeout code and received input from a `futures::mpsc::channel`
- And one for listening to i3's UNIX socket, the domain of`i3ipc-rs`

And thus, I am writing a library intended to be the async version of `i3ipc`, because all it takes is the entire ecosystem to be rewritten in async IO for me to finally be able to do all of this in one thread.
