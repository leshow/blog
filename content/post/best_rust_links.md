---
title: "After 5 years, my favorite Rust links"
date: 2020-05-29T15:51:41-04:00
draft: true
---

I've been following and writing Rust for a long time now, since just pre-1.0. I'm also the type of person that keeps a fairly organized set of bookmarks, so I've collected a lot of wonderful resources over the years.

Given we just recently had the 5 year anniversary of 1.0, I'd like to some of share my favourite Rust links of the last 5 years. The list is probably skewed towards the async ecosystem and type system things, just because that's the sort of stuff that I follow.

## General Rust

- [Official Rust Book](https://doc.rust-lang.org/book/title-page.html)

  Still the best way to start learning Rust IMO. The official Rust book is a wonderful free resource.

- [Rust cookbook](https://rust-lang-nursery.github.io/rust-cookbook/)
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/index.html)
- [API guidelines](https://rust-lang.github.io/api-guidelines/)
- [Rust Reference](https://doc.rust-lang.org/reference/introduction.html)
- [Abstraction without overhead: Traits](https://blog.rust-lang.org/2015/05/11/traits.html)
- [Learning Rust with entirely too many linked lists](https://rust-unofficial.github.io/too-many-lists/)

  This is a great introduction to a lot of language features and probably my first real intro to unsafe

- [Rust cheat sheet](https://cheats.rs/)

  For quick look-ups, it's hard to beat the rust cheat sheet

- [Rustlings](https://github.com/rust-lang/rustlings)

  these are small exercises you can use to get started in Rust

- [Rust performance pitfalls](https://llogiq.github.io/2017/06/01/perf-pitfalls.html)

- [Error Handling in Rust](https://blog.burntsushi.net/rust-error-handling/) (burntsushi)

- [Many types of Code Reuse in Rust](http://cglab.ca/~abeinges/blah/rust-reuse-and-recycle/) (gankra)
- [Allocations in Rust](https://speice.io/2019/02/understanding-allocations-in-rust.html)

  If you want to learn about smart pointers and heap allocation, I really like this one

- [Zero Cost Abstractions](https://boats.gitlab.io/blog/post/zero-cost-abstractions/) (boats)

This should really be mandatory reading for any Rust dev:

- [Interior mutability in Rust](https://ricardomartins.cc/2016/06/08/interior-mutability) (ricardo martins)

  [int. mut. part 2](https://ricardomartins.cc/2016/06/25/interior-mutability-thread-safety)
  [int mut. part 3](https://ricardomartins.cc/2016/07/11/interior-mutability-behind-the-curtain)

- [Idiomatic conversions in Rust](https://ricardomartins.cc/2016/08/03/convenient_and_idiomatic_conversions_in_rust) (ricard martins)
- [Another look at the pinning API](https://boats.gitlab.io/blog/post/rethinking-pin/) (boats)
- [Intersection impls](http://smallcultfollowing.com/babysteps/blog/2016/09/24/intersection-impls/) (niko matsakis)

  A wonderful blog series on specialization,
  follow ups: [here](http://smallcultfollowing.com/babysteps/blog/2016/09/29/distinguishing-reuse-from-override/) and [here](http://smallcultfollowing.com/babysteps/blog/2016/10/24/supporting-blanket-impls-in-specialization/)
  both by Niko

- [Associated type constructors](http://smallcultfollowing.com/babysteps/blog/2016/11/02/associated-type-constructors-part-1-basic-concepts-and-introduction/) (niko)

  Follow ups: [here](http://smallcultfollowing.com/babysteps/blog/2016/11/03/associated-type-constructors-part-2-family-traits/), [here](http://smallcultfollowing.com/babysteps/blog/2016/11/04/associated-type-constructors-part-3-what-higher-kinded-types-might-look-like/), [here](http://smallcultfollowing.com/babysteps/blog/2016/11/09/associated-type-constructors-part-4-unifying-atc-and-hkt/)

- [Join your threads](https://matklad.github.io/2019/08/23/join-your-threads.html) (matklad)

- Amos has a great set of articles:

  - [Half hour to learn Rust](https://fasterthanli.me/articles/a-half-hour-to-learn-rust)
  - [Getting in and out of trouble with Rust Futures](https://fasterthanli.me/articles/getting-in-and-out-of-trouble-with-rust-futures)
  - [Working with Strings in Rust](https://fasterthanli.me/articles/working-with-strings-in-rust)
  - [Small Strings in Rust](https://fasterthanli.me/articles/small-strings-in-rust)
  - [Dynamic linker murder mystery](https://fasterthanli.me/articles/a-dynamic-linker-murder-mystery)

- [Rayon data Parallelism in Rust](https://smallcultfollowing.com/babysteps/blog/2015/12/18/rayon-data-parallelism-in-rust/)

## Async IO

- [Tokio mini-redis tutorial](https://tokio.rs/tokio/tutorial)
- [Async Book (unfinished)](https://rust-lang.github.io/async-book/)

  Unfortunately not complete, but still a great resource

- [Tokio crash course](https://www.snoyman.com/blog/2019/12/rust-crash-course-09-tokio-0-2)

  Snoyman did a wonderful 'crash course' on tokio that has lots of good content, and a question/answer style that I like quite a bit

- [Tokio internals](https://cafbit.com/post/tokio_internals/)
- [On the tokio scheduler](https://tokio.rs/blog/2019-10-scheduler/)
- [ringbahn](https://boats.gitlab.io/blog/post/ringbahn/) (boats)

  So many good blogs from boats, hard to pick just a few

- [Async Destructors](https://boats.gitlab.io/blog/post/poll-drop/) (boats)
- [Async Interviews](http://smallcultfollowing.com/babysteps/blog/2020/03/10/async-interview-7-withoutboats) (niko)

  All of them are great

These two are a little outdated now and describe the Futures 0.1 trait, but are still interesting to see motivations:

- [Zero cost Futures in Rust](https://aturon.github.io/blog/2016/08/11/futures/) (aturon)
- [Designing Futures for Rust](https://aturon.github.io/blog/2016/09/07/futures-design/) (aturon)

## Wasm

- [Rust wasm book](https://rustwasm.github.io/docs/book/)
- [Rust wasm-bindgen](https://rustwasm.github.io/docs/wasm-bindgen/)

## Embedded

- [Rust Embedded book](https://docs.rust-embedded.org/book/)
- [Rust Discovery book](https://docs.rust-embedded.org/discovery/)
- [Writing an OS in Rust](https://os.phil-opp.com/) (Philipp Oppermann)
- [Real Time Interrupt Driven Concurrency (book)](https://rtic.rs/0.5/book/en/preface.html)

  and [this](http://blog.japaric.io/brave-new-io/) accompanying blog post

## Unsafe

- [Rustonomicon](https://doc.rust-lang.org/nomicon/index.html)
- [Two memory bugs from ringbahn](https://without.boats/blog/two-memory-bugs-from-ringbahn/) (boats)
- [Unsafe Abstractions](https://smallcultfollowing.com/babysteps/blog/2016/05/23/unsafe-abstractions/) (niko)
