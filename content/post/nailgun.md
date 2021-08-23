---
title: "nailgun: DNS benchmarking tool"
date: 2021-08-22T09:55:12-04:00
draft: false
---

[nailgun](https://github.com/leshow/nailgun) is a small hobby project I've been working on sparsely for a few months, it's a cli DNS benchmarking tool heavily inspired by [flamethrower](https://github.com/DNS-OARC/flamethrower) but written in Rust with tokio. I've been working at the intersection of DNS & Rust for a little while and this is something I've been pushing along in my free time.

Rest assured, it has no real reason for existing just yet other than I felt like writing it so if you're looking for a quality DNS performance testing client then you should probably still use flamethrower. But, if you want to help make `nailgun` better then I'm open to PRs, issues, or suggestions. I feel I've got things in a "good enough" state that I can release the code and continue to develop it in the open. My hope is that at the very least it's another small-ish tokio project that people can look at for clues when building their own projects. If I haven't dissuaded you yet, it's available [here](https://github.com/leshow/nailgun) on github.

## Design

By default, `nailgun` will run one tokio worker thread and one traffic generator task, with both values being configurable via cli args. Each generator creates a `TcpStream` or `UdpSocket` and a `Store`. `Store` is a `parking_lot::Mutex` protected chunk of memory storing the available ids and a hashmap representing the current queries that are in-flight, mapped by their id.

```rust
pub struct Store {
    ids: VecDeque<u16>,
    in_flight: FxHashMap<u16, QueryInfo>, // stores time sent & buffer len
}
```

DNS is an ancient protocol and its ids are `u16` values, so there is an upper limit on the number of in-flight messages any generator can have at one time (2^16 - 1 = 65535). I considered alternate designs without the mutex but in practice it seemed plenty fast enough, and an async mutex is not necessary here because a lock is [never held across an await point](https://tokio-rs.github.io/tokio/doc/tokio/sync/struct.Mutex.html#which-kind-of-mutex-should-you-use).

On startup, the entire range of u16 values are generated and randomly shuffled, then put in a `VecDeque` so that new messages can pop ids from the front and received ids pushed to the back. Since nailgun doesn't care too much about the contents of the message (yet) they are not decoded, we just peak in the header and mask off the bits necessary to get things like the id & rcodes. This strategy means that we're only looking at the lower 4 bits of the rcode, ignoring EDNS. Maybe this is something to expand on later. If you want more information about what a DNS header or message looks like, I have some 2000's era websites for you! [networksorcery](http://www.networksorcery.com/enp/default.htm) and [tcp/ip guide](http://www.tcpipguide.com/free/t_DNSMessageHeaderandQuestionSectionFormat.htm) both have great DNS sections. The relevant RFCs are also good but there are many and they are pretty dry.

In addition to this the store, generators start 2 tasks: one for creating & sending all the DNS messages, and one for handling timed out queries (look through the in-flight hashmap and clear anything above a certain age, returning cleared ids to the queue).

The generator's main loop is a `tokio::select!` future with 3 branches:

- read a message from UDP or TCP
- every second-- gather up all the stats from this generator & send via mpsc channel to another task that will aggregate/log
- listen for shutdown signal or any child tasks handles returning. Shutdown portions lifted from [mini-redis](https://github.com/tokio-rs/mini-redis)

If this is all clear as mud, I've done my best to represent it visually. The blue boxes are tokio tasks, each with a reference to the `Store`, and showing the main futures they execute in green:

![generator](/nailgun/nailgun_generator.jpg)

## Ecosystem

There are other elements at play here, like query generation, logging, and rate limiting, but I think I've covered the interesting bits. I'd be remiss if I didn't mention the many awesome crates in the Rust ecosystem that make something like this application doable in a few thousand lines:

- `tokio` obviously

- `trust-dns-proto` is awesome for anything DNS related, they have first-class types for parsing and encoding dns messages

- `tracing` takes care of logging, I use it in pretty much every project these days. nailgun is set up to output structured JSON or unstructured, and optionally write to file. There is some friction if you want to change tracing options at startup based on user input (cli flags in this case) however, the code can be a bit repetitive because tracing lifts its config to the type level ([Layered](https://docs.rs/tracing-subscriber/0.2.20/tracing_subscriber/layer/struct.Layered.html) is pretty much an HList afaict, reminds me a lot of tower's [Stack](https://github.com/tower-rs/tower/blob/master/tower-layer/src/stack.rs#L6)). There may be some better way to do this but I have not found it.

- `governor` the nailgun repo still has remnants of a few token bucket yak shaves before I decided to just use this and everything "just worked".

- `clap` for arg parsing, although I had a few issues where clap couldn't express the same parameter types as flamethrower meaning I had to diverge a bit. In flamethrower, you can pass `-g randomlabel lblsize=10 lblcount=4` where the first arg to `-g` is one of a set of variants, then the others are variables passed to it. As far as I can tell there is no way to do that in `clap`. You can have a subcommand but not behind an arg, and you can parse key-value pairs but not like this.

- notable mentions: `anyhow`/`parking_lot`/`async_trait`

## Current Limitations & Conclusion

There's a couple things I'd like to improve. Firstly, the TCP traffic generator opens up a single `TcpStream`; this probably needs to change so that it's more representative of the real world. More investigation needs to be done with running additional concurrent traffic generators there currently isn't much of a performance difference. The statistics gathering/logging is a bit messy, and I'm sure there is room for better abstractions in a few places. So that's pretty much everything :)

I'd also like to have the default traffic generating mechanism rate-limit itself once the downstream starts to return some threshold of timeouts, perhaps making this configurable. Currently, if you don't pass a QPS (`-Q` queries per second) parameter then nailgun will just continue to send traffic as fast as possible. Also, there are some features flamethrower has that would be nice to add to nailgun too, like its ability to construct a ["QPS flow"](https://github.com/DNS-OARC/flamethrower#dynamic-qps-flow).

Until next time!
