---
title: "A_simple_tokio_proto"
date: 2019-03-22T14:48:24-04:00
draft: true
---

A friend of mine write a time-tracking library for the i3 tiling window manager called [i3-tracker](https://github.com/danbruce/i3-tracker-rs) a little while ago. It's completely synchronous, and that always bothered me for no good reason. The idea of spawning a new thread just to do a timeout and potentially having those pile up unbounded-- while likely never to account for anything more than a momentary stutter, screamed out at me for premature optimization. So, I did what any individual immersed in Rust of the times would do: I attempted to re-write it all in tokio.

Tokio is super cool, it's also has a rep for being unapproachable. I'm not a systems developer, I write code for the web. So while concepts like asynchronicity aren't ne to me, a lot of the accompanying ideas are. Lack of implicit runtimes, GC, the whole bag. Still, I managed to fork `i3-tracker` and get everything running in a single thread without too much hassle. Afterwards, I proudly went to look at my achievement in htop only to find 2 threads running:

- One for tokio that had all my futures + timeout code and received input from a `futures::mpsc::channel`
- And one for listening to i3's UNIX socket, provided by `i3ipc-rs`

And thus, I decided to rewrite `i3ipc-rs` using tokio as well, for an even further useless optimization. I give you: [tokio-i3ipc](https://github.com/leshow/tokio-i3ipc) The reality is I find the whole Rust async ecosystem fascinating and I wanted to learn more. I doubt many will use the library (though I'm more than happy if they do). But perhaps I can help those trying to get in to the tokio ecosystem with a bit of a guide. Something I wish I had when I started.
