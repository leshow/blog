---
title: "A look at tokio 1.0 API Changes"
date: 2020-12-28T10:19:23-05:00
draft: false
---

I've been working in the Rust space for about a year now using tokio & async/await with DNS. The result of this work is a sizeable from-scratch tokio server using 0.2 (that's now in production-- yay! hopefully I can share more about this later). As a result, I've gotten to know the UDP API of tokio quite well, and have even got to submit a few PRs, nothing groundbreaking mind you. I'd like to highlight some interesting changes that have happened in tokio's API between `0.2` and `0.3`/`1.0`. These are all externally facing changes, and will tend to focus on UDP since that's what I know. I wouldn't consider myself knowledgeable enough about the internals of what prompted these changes to dig into what's going on under the hood, but I'll do my best to point to relevant issues or PRs so you can read more. And if this prompts some discussion which further elucidates some of the details; all the better.

With that out of the way, let's tuck in...

## Types are no longer &mut self

As a result of [#2779](https://github.com/tokio-rs/tokio/issues/2779) `net` types in tokio (`UdpSocket`, `TcpStream`, etc) no longer require `&mut self` to `recv`/`send` (or `read`/`write` in `TcpStream`'s case). This is a great thing for writing code that needs to concurrently read and write on the same socket. If you want to read/write from the same task, you can just use a regular `&UdpSocket` reference, and if you want to read in one task and write in another, a simple `Arc<UdpSocket>` will do. `TcpStream` keeps the `split` method, but under the hood it's doing just what I mentioned.

### Before

Now, for concurrent send/recv, I'm not sure how others used to handle this case, but I always would set up a channel and create a dedicated "sender" task, then I'd `recv`, do some work, and send the response back over the channel. Putting that all together with `split` looked more or less like this:

```rust
async fn run() -> Result<()> {
    let udp = UdpSocket::bind("0.0.0.0:8080").await?;
    // call split so we can give ownership of 'half' to one task
    let (mut r, mut s) = udp.split();

    let (mut tx, mut rx) = mpsc::channel::<(Vec<u8>, SocketAddr)>(1_000);

    tokio::spawn(async move {
        while let Some((msg, addr)) = rx.recv().await {
            let len = s.send_to(&msg, &addr).await.unwrap();
            println!("{:?} bytes sent", len);
        }
    });

    let mut buf = [0; 1024];
    loop {
        let (len, addr) = r.recv_from(&mut buf).await?;
        println!("{:?} bytes received from {:?}", len, addr);
        tx.send((buf[..len].to_vec(), addr)).await.unwrap();
    }
}
```

You can see here `split` gets around the `&mut self` property of `recv` and `send`. The split api let's us get each "half" that can be moved into it's own fully owned task, we can then concurrently `send` and `recv` afterwards.

### After

Now, with the above changes in 1.0, `split` is gone for `UdpSocket` and we can just use regular types from the std lib to accomplish the same task. If you're using `TcpStream` or `UnixStream`, you can choose to use `split`/`into_split` or `&`/`Arc`. For these types, I'd recommend split if you only need two "halves" and use the std types if you want to use more than two.

```rust
async fn main() -> io::Result<()> {
    let sock = UdpSocket::bind("0.0.0.0:8080").await?;
    // Arc instead of split, make two refs
    let r = Arc::new(sock);
    let s = r.clone();
    let (tx, mut rx) = mpsc::channel::<(Vec<u8>, SocketAddr)>(1_000);

    tokio::spawn(async move {
        while let Some((bytes, addr)) = rx.recv().await {
            let len = s.send_to(&bytes, addr).await.unwrap();
            println!("{:?} bytes sent", len);
        }
    });

    let mut buf = [0; 1024];
    loop {
        let (len, addr) = r.recv_from(&mut buf).await?;
        println!("{:?} bytes received from {:?}", len, addr);
        tx.send((buf[..len].to_vec(), addr)).await.unwrap();
    }
}
```

Another thing about `&self` methods is that we can `send`/`recv` from BOTH tasks, whereas before there was a dedicated "sending" and "receiving" half. You could also imagine a more complicated setup where we give a reference to more than two tasks.

Note that in both before/after cases here, we could easily `tokio::spawn` in the loop and pass in a `clone`d `Sender` in order to run each message in it's own task (if we were doing something actually useful). That way we're not doing anything in our read loop but pulling data off the socket.

## Poll API

If you want to use a `Stream` implementation for `UdpSocket`, your previous option was pretty much just `UdpFramed` in `tokio-util`. If you haven't used any of the util crates, you write a "decoder" which understands how to transform a buffer into a "frame" of your chosen type.

```rust
pub trait Decoder {
    type Item;
    type Error: From<Error>;
    fn decode(
        &mut self,
        src: &mut BytesMut
    ) -> Result<Option<Self::Item>, Self::Error>;
    ...
}
```

This API has remained much the same in the new `tokio-util`, although the internals of how it pulls down a frame has changed. Under the hood, `Decoder` used a set of methods prefixed with `poll_`, these are methods that are a mirror of their async counterparts, but that return `Poll`. These methods have now been committed to the public API. In other words, `UdpSocket` now supports a `Poll` returning methods for `recv`/`recv_from`/`send`/`send_to`.

So now we have an alternative to `UdpFramed` to write manual `Stream` impls and a committed API for `Future` impls. This is a welcome addition if you don't want to be tied to other parts of the tokio ecosystem or you just need the extra flexibility.

If you're unfamiliar with this `poll_` / `async` dichotomy, count yourself as lucky. It's nice that most users will only ever write `async` methods and never manually implement a `Future` or `Stream`. But sometimes in library code you need to do these things, and so tokio provides the building blocks, as a low level library, for both an `async` and `Context` receiving-- `Poll` returning methods now for each read/write action.

To make things more concrete, `UdpSocket` has the async method `recv`:

```rust
pub async fn recv(&self, buf: &mut [u8]) -> Result<usize>
```

If you're in async/await land this is great, use these methods. It's just when you're stuck writing `Stream` or `Future` impls that these methods don't do you any good. Look at the type signature of a `Future`:

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

`Stream` has a similar style to this. If you've got async methods, calling them from `poll` can be difficult and often produce bugs because you can easily end up creating a new future _on every invocation of `poll`_ rather than holding on to a single future and driving it to completion.

In short, we need a `Poll` returning method that takes `Context`. This is where the `poll_*` variants come in:

```rust
pub fn poll_recv(&self, cx: &mut Context<'_>, buf: &mut ReadBuf<'_>) -> Poll<Result<()>>
```

This is a bit off-topic, but you can quickly turn a `poll_*` method into an async fn with [`poll_fn`](https://docs.rs/futures/0.3.8/futures/future/fn.poll_fn.html); and indeed, the async methods used to be written this way.

```rust
poll_fn(|cx| self.poll_recv_from(cx, buf)).await
```

However, there is an edge-case to be aware of here, and it's outlined in the docs on the `poll_*` methods:

```rust
/// Note that on multiple calls to a `poll_*` method in the send direction, only the
/// `Waker` from the `Context` passed to the most recent call will be scheduled to
/// receive a wakeup.
```

and on recv:

```rust
/// Note that on multiple calls to a `poll_*` method in the recv direction, only the
/// `Waker` from the `Context` passed to the most recent call will be scheduled to
/// receive a wakeup.
```

You should not try to _concurrently_ use either the `poll_recv` or `poll_send` methods _in the same direction_, and if you do not observe this advice-- expect weirdness.

## ReadBuf

You may have noticed the `poll_recv` method takes a `ReadBuf` type and not a `&mut [u8]`, this is a new type that `AsyncRead` and `AsyncWrite` also use in order to read/write into possibly uninitialized memory [#2716](https://github.com/tokio-rs/tokio/issues/2716). `ReadBuf` is a low level type that uses `MaybeUninit<u8>` under the hood and tracks what parts of the buffer have been filled/initialized. Why would we want to do this? The main issue with stack buffers like `[u8; 1024]`, as I understand it, is that the buffer must always be zero-initialized. This is a potentially costly operation, you have to allocate and zero out all of that memory. Because this is a divergence from the std types, there is a corresponding RFC to merge `ReadBuf` [into std](https://github.com/rust-lang/rust/issues/78485).

## tokio-stream

`tokio-stream` is a new crate that moves `tokio::stream::Stream` & related traits are out and into it's own crate. There's also a bunch of helpful `tokio_stream::wrappers` which can turn a `TcpListener` or `UnixListener` into a `Stream`. Other wrappers include `Interval` for working with a stream of timers. These wrapper types often use the `poll_*` API's I mentioned earlier.

## Sundries

- `UdpSocket` and it's Tcp counterparts have gotten `async` readiness checking methods [#3138](https://github.com/tokio-rs/tokio/pull/3138).
- Everything that uses `SocketAddr` now takes it by value, since the type is `Copy` we don't need the extra layer of indirection. Old tokio API's would often take `&SocketAddr`. async methods are still generic `<T: ToSocketAddrs>`.
- `broadcast::Sender<_>` uses `send` instead of `broadcast` now
- `watch::Receiver<_>` uses `changed` now instead of `recv`, and `changed` only shows you readiness. You need to explicitly `borrow()` and `clone()` yourself now if you want a fresh `T`
- `acquire()` for `Semaphore` returns a `Result<SemaphorePermit, _>` now instead of just `SemaphorePermit`
- `time::delay_until` and `time::delay` have been renamed to `time::sleep` to be more std-like in naming
- Is there more missing here? Send me a message and let me know!

## Something I'd like to see in the future

I'd like to see some way to bound on a type that can `send`/`recv` in the same way we can `<T: AsyncRead + AsyncWrite>`. I think this probably won't be added until there is a zero-cost way to make async methods in traits, and even then, I'm not sure if the tokio team even desires that behaviour. Personally, I think it would make writing `Framed` impls much more generic and allow a function to take both a `UdpSocket` or `Arc<UdpSocket>` generically. This would help when you want to make a `Decoder` for a socket that is shared among tasks concurrently.

## Wrapping up

As an early user of tokio and the futures ecosystem, I remember the slog of futures 0.1 and tokio 0.1, seeing that evolve into the more mature `std::future` and tokio 0.2 and now 1.0 has been fascinating. It's hard to believe it's been something like 4 years since this all started. I'm happy that most will not have to experience a manual `Future` impl or working with nested combinators. I'm even happier, as a user of tokio, that they've decided to commit to this API for a good length of time. That keeps some of the burden off of application developers having to "keep up" with everything that's going on in the ecosystem. Not everyone has the desire to follow all of the PRs or subreddits & message boards. I'm not speaking for myself, personally I can't get enough, but I understand that's not how others want to spend their free time.

Anyway, congrats tokio team, you've done a wonderful job!
