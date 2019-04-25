---
title: "Serialization: Haskell & Rust"
date: 2019-03-20T22:52:17-04:00
draft: true
---

I recently published my first Haskell [package](https://hackage.haskell.org/package/i3ipc-0.1.0.0). While I'm almost sure I have no users yet, it was a great experience. I wrote a slim wrapper around i3's IPC, basically doing the the grunt work of writing all the necessary records for serialization, and a couple hundred lines of functions (mostly in IO) to interact with the common use cases (subscribing to events, retrieving data).

It probably wasn't worth the aggravation; my estimation is the cross section of Haskell users that also use tiling window managers would show a strong overlap with XMonad, not i3, but here we are. At least it gave an interest opportunity for comparison with a library in my other favorite language: Rust.

The i3 package I wrote for Haskell was a warm up in a lot of ways for writing one in Rust using tokio. I've [completed that also](https://crates.io/crates/tokio-i3ipc) As a result, I've re-written all of the i3 IPC types a twice now in both Haskell and Rust. Once using `aeson`, then with `serde`. Since Haskell and Rust share a lot of similarities in their type systems; typeclasses/traits, deriving/derive, sum types/enum, etc. I thought it would be cool to take a look at these two libraries that fill the same niche in different languages.

## Approach to Field Names

### Haskell

I'm not sure if aeson was one of the first libraries to really popularize the whole auto-derived instances thing, but it certainly _feels_ like it, it was first released in 2011. The `DeriveGeneric` extension it relies upon was first released in GHC 7.2 in November of 2011.

In aeson, defining a data type and deriving looks a bit magical in that you need to perform the correct incantations:

```haskell
{-# LANGUAGE DeriveGeneric, BangPatterns #-}

import GHC.Generics

data WorkspaceEvent = WorkspaceEvent {
    wrk_change :: !WorkspaceChange
    , wrk_current :: !(Maybe Node)
    , wrk_old :: !(Maybe Node)
} deriving (Eq, Generic, Show)
```

A couple things to note here. Firstly, the bang (!) patterns annotate the field as strict rather than lazy, they aren't required, but I didn't see any point in having lazy fields in my data structures. Second, records in Haskell are a bit ornery on account of field names polluting the global namespace. As a result, I prefixed all of the field names with a keyword and underscore. The issue now, for the purposes of derivation, is that my field names no longer match the JSON keys in the structure; so the completely automatic deriving won't provide an instance that matches my real world structure.

Aeson's solution for that:

```haskell
instance ToJSON WorkspaceEvent where
    toEncoding = genericToEncoding defaultOptions { fieldLabelModifier = drop 4 }

instance FromJSON WorkspaceEvent where
    parseJSON = genericParseJSON defaultOptions { fieldLabelModifier = drop 4 }
```

With this, when serializing to/from JSON, aeson will drop the first 4 characters and the names will again match.

If I didn't have prefixed field names, creating the instances can be completely automated:

```haskell
{-# LANGUAGE DeriveGeneric, BangPatterns #-}

import GHC.Generics

data WorkspaceEvent = WorkspaceEvent {
    change :: !WorkspaceChange
    , current :: !(Maybe Node)
    , old :: !(Maybe Node)
} deriving (Eq, Generic, Show, FromJSON, ToJSON)
```

This makes use of the generic machinery provided by the language extension `DeriveGeneric`, this allows us to get a (surprise) generic representation of data structures that libraries can plug in to to write instances. `Eq` and `Show` are part of the built in derivable typeclasses in GHC (others include `Ord`, `Bounded`, `Read`).

### Rust

Serde takes a similar approach to deriving instances, it had it's first "pre-release" in 2016, and shipped 1.0 April of 2017 after rustc stable added support for the procedural macros that enable it's own derive mechanism (I believe this was Rust 1.15). In Rust, struct fields are locally namespaced so the antecedent keyword used above isn't required. In Rust the derive mechanism is a special kind of macro called a proc macro. It looks like this:

```rust
use serde::{Serialize, Deserialize};

#[derive(Deserialize, Serialize, PartialEq, Debug, Clone)]
pub struct WorkspaceData {
    pub change: WorkspaceChange,
    pub current: Option<reply::Node>,
    pub old: Option<reply::Node>,
}
```

`Serialize` and `Deserialize` come from serde, the other traits I'm deriving are part of Rust's std library and are among the half dozen derivable traits for a struct. In this regard Rust is very similar to Haskell. An interesting thing to note is Rust's trait to describe equality is split into `PartialEq` and it's supertrait `Eq`. The delineation helps when modelling floating point numbers, of which `PartialEq` is a member.

You may have noticed I didn't need to make any modifications to get the names on my struct to match the actual JSON, but if I did, I could annotate the individual members rather than writing a minimal implementation:

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
