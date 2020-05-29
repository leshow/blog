---
title: "After 5 years, my favorite Rust links"
date: 2020-05-29T15:51:41-04:00
draft: true
---

I've been following and writing Rust for a long time now, since just pre-1.0. I'm also the type of person that keeps a fairly organized set of bookmarks, so I've collected a lot of wonderful resources over the years.

Given we just recently had the 5 year anniversary of 1.0, I'd like to some of share my favourite Rust links of the last 5 years.

## General Rust

- Official Rust Book - https://doc.rust-lang.org/book/title-page.html

  Still the best way to start learning Rust IMO. The official Rust book is a wonderful free resource.

- Rust cookbook - https://rust-lang-nursery.github.io/rust-cookbook/
- Rust by Example - https://doc.rust-lang.org/rust-by-example/index.html
- API guidelines - https://rust-lang.github.io/api-guidelines/
- Rust Reference - https://doc.rust-lang.org/reference/introduction.html
- Abstraction without overhead: Traits - https://blog.rust-lang.org/2015/05/11/traits.html
- Learning Rust with entirely too many linked lists - https://rust-unofficial.github.io/too-many-lists/

  This is a great introduction to a lot of language features and probably my first real intro to unsafe

- Rust cheat sheet - https://cheats.rs/

  For quick lookups, it's hard to beat the rust cheat sheet

- Rustlings - https://github.com/rust-lang/rustlings

  these are small exercises you can use to get started in Rust

- Rust performance pitfalls - https://llogiq.github.io/2017/06/01/perf-pitfalls.html

- Many types of Code Reuse in Rust - http://cglab.ca/~abeinges/blah/rust-reuse-and-recycle/
- allocations in Rust - https://speice.io/2019/02/understanding-allocations-in-rust.html

  If you want to learn about smart pointers and heap allocation, I really like this one

- Zero Cost Abstractions - https://boats.gitlab.io/blog/post/zero-cost-abstractions/
- Interior mutability in Rust - https://ricardomartins.cc/2016/06/08/interior-mutability

  [part 2](https://ricardomartins.cc/2016/06/25/interior-mutability-thread-safety)

- Idiomatic conversions in Rust - https://ricardomartins.cc/2016/08/03/convenient_and_idiomatic_conversions_in_rust
- Another look at the pinning API - https://boats.gitlab.io/blog/post/rethinking-pin/
- Intersection impls - http://smallcultfollowing.com/babysteps/blog/2016/09/24/intersection-impls/

  A wonderful blog series on specialization,
  follow ups: [here](http://smallcultfollowing.com/babysteps/blog/2016/09/29/distinguishing-reuse-from-override/) and [here](http://smallcultfollowing.com/babysteps/blog/2016/10/24/supporting-blanket-impls-in-specialization/)

- Associated type constructors - http://smallcultfollowing.com/babysteps/blog/2016/11/02/associated-type-constructors-part-1-basic-concepts-and-introduction/

  Follow ups: [here](http://smallcultfollowing.com/babysteps/blog/2016/11/03/associated-type-constructors-part-2-family-traits/), [here](http://smallcultfollowing.com/babysteps/blog/2016/11/04/associated-type-constructors-part-3-what-higher-kinded-types-might-look-like/), [here](http://smallcultfollowing.com/babysteps/blog/2016/11/09/associated-type-constructors-part-4-unifying-atc-and-hkt/)

## Async IO

- Async Book (unfinished) - https://rust-lang.github.io/async-book/

  Unfortunately not complete, but still a great resource

- Tokio crash course - https://www.snoyman.com/blog/2019/12/rust-crash-course-09-tokio-0-2

  Snoyman did a wonderful 'crash course' on tokio that has lots of good content, and a question/answer style that I like quite a bit

- Tokio internals - https://cafbit.com/post/tokio_internals/
- On the tokio scheduler - https://tokio.rs/blog/2019-10-scheduler/
- ringbahn - https://boats.gitlab.io/blog/post/ringbahn/

  So many good blogs from boats, hard to pick just a few

- Async Destructors - https://boats.gitlab.io/blog/post/poll-drop/
- Async Interviews - http://smallcultfollowing.com/babysteps/blog/2020/03/10/async-interview-7-withoutboats/

  All of them are great

## Wasm

- Rust wasm book - https://rustwasm.github.io/docs/book/
- Rust wasm-bindgen https://rustwasm.github.io/docs/wasm-bindgen/

## Embedded

- Rust Embedded book - https://docs.rust-embedded.org/book/
- Rust Discovery book - https://docs.rust-embedded.org/discovery/
- Writing an OS in Rust - https://os.phil-opp.com/
- Real time for the masses - https://japaric.github.io/rtfm5/book/en/preface.html

  and [this](http://blog.japaric.io/brave-new-io/) accompanying blog post

## Unsafe

- Rustonomicon - https://doc.rust-lang.org/nomicon/index.html
