---
title: "Tokio 1.0 API Changes"
date: 2020-12-24T15:49:23-05:00
draft: true
---

I've been working in the Rust space for about a year now using tokio & async/await in the DNS space. The result of this work is a sizeable from-scratch tokio server using 0.2 (that's now in production-- yay! hopefully I can share more about this later). As a result, I've gotten to know the UDP API of tokio quite well, and have even got to submit a few patches. Nothing groundbreaking mind you. But I'd like to highlight some interesting changes I've observed in tokio's API between `0.2` and `0.3`/`1.0`. These are all externally facing changes, and will tend to focus on UDP since that's what I know. I wouldn't consider myself knowledgeable enough about the internals of what prompted these changes to dig into what's going on under the hood, but I'll do my best to point to relevant issues or PRs. And if this prompts some discussion which further elucidates some of the details; all the better.

With that out of the way, let's tuck in...

## Types are no longer &mut self (bye bye split)

As a result of [#2779](https://github.com/tokio-rs/tokio/issues/2779) `UdpSocket` and `TcpStream` no longer require `&mut self` to `recv`/`send` (or `read`/`write` in `TcpStream`'s case). This is a great thing for writing code that needs to concurrently read and write on the same socket, as we can skip the whole `split` api and just use `Arc`.

### Before

Now, I'm not sure how others used to handle this case, but I always would set up a channel and create a dedicated "sender" task, then I'd `recv`, do some work, and send the response back over the channel. Putting that all together with `split` that looked more or less like this:

```rust
async fn run() -> Result<()> {
    let udp = UdpSocket::bind("0.0.0.0:8080").await?;
    // call split so we can give ownership of 'half' to one task
    let (mut udp_recv, mut udp_send) = udp.split();

    let (mut tx, mut rx) = mpsc::channel::<(Vec<u8>, SocketAddr)>(1_000);

    tokio::spawn(async move {
        while let Some((msg, addr)) = rx.recv().await {
            let len = s.send_to(&msg, &addr).await.unwrap();
            println!("{:?} bytes sent", len);
        }
    });

    let mut buf = [0; 1024];
    loop {
        let (len, addr) = udp_recv.recv_from(&mut buf).await?;
        println!("{:?} bytes received from {:?}", len, addr);
        tx.send((buf[..len].to_vec(), addr)).await.unwrap();
    }
}
```

You can see here `split` gets around the `&mut self` property of `recv` and `send`. The split api let's us get each "half" that can be moved into it's own fully owned task, we can then concurrently `send` and `recv` afterwards.

### After

Now, with the above changes in 1.0, `split` is gone and we can just use regular types from the std lib to accomplish the same task.

```rust
async fn main() -> io::Result<()> {
    let sock = UdpSocket::bind("0.0.0.0:8080").await?;
    // look ma-- no split!
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

Another great thing about `&self` methods is that we can `send`/`recv` from BOTH tasks, whereas before there was a dedicated "sending" and "receiving" half. You could also imagine a more complicated setup where we give a reference to more than two tasks.

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

This API has remained much the same in the new `tokio-util`, although the internals of how it pulls down a frame has changed. Under the hood, `Decoder` used a set of methods prefixed with `poll_`, these are methods that are a mirror of their async counterparts, but that return `Poll`. In other words, `UdpSocket` now supports a `Poll` API, it has public methods for `recv`/`recv_from`/`send`/`send_to` that don't use `async` and instead return the `Poll` type.

So now we have another way to write manual `Stream`/`Future` impls, which is a welcome addition if you don't want to be tied to other parts of the tokio ecosystem or you just need the extra flexibility.

If you're unfamiliar with this `poll_` / `async` dichotomy, count yourself as lucky. It's nice that most users will only ever write `async` methods and never manually implement a `Future` or `Stream`. But sometimes in library code you need to do these things, and so tokio provides the building blocks, as a low level library, for both an `async` and non-async (`Context` receiving-- `Poll` returning) method now for each read/write action.

To make things more concrete, `UdpSocket` has the async method `recv` (lifetimes removed):

```rust
pub async fn recv(&self, buf: &mut [u8]) -> Result<usize>
```

If you're in async/await land this is great, use these methods. It's just when you're stuck writing `Stream` or `Future` impls that these methods don't do you any good. You need to be able to pass through `Context` and use `Poll`. This is where the `poll_*` variants come in:

```rust
pub fn poll_recv(&self, cx: &mut Context<'_>, buf: &mut ReadBuf<'_>) -> Poll<Result<()>>
```

With these methods exposed, you're free to write your own `Stream`/`Future` impls. This is a bit off-topic, but you can quickly turn a `poll_*` method into an async fn with [`poll_fn`](https://docs.rs/futures/0.3.8/futures/future/fn.poll_fn.html)

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

You may have noticed the `poll_recv` method exposes a `ReadBuf` type, this is a new type that `AsyncRead` and `AsyncWrite` use in order to read/write into possibly uninitialized memory [#2716](https://github.com/tokio-rs/tokio/issues/2716). `ReadBuf` is a low level type that uses `MaybeUninit<u8>` under the hood and tracks what parts of the buffer have been filled/initialized. The main issue with static buffers like `[u8; 1024]`, as I understand it, is that the buffer must always be zero-initialized. This is a potentially costly operation, imagine on every read you have to allocate and zero out all of that memory. Because this is a divergence from the std types, there is a corresponding RFC to merge `ReadBuf` [into std](https://github.com/rust-lang/rust/issues/78485).

## Smaller changes

- `UdpSocket` and it's Tcp counterparts have gotten `async` readiness checking methods [#3138](https://github.com/tokio-rs/tokio/pull/3138).
- Everything that uses `SocketAddr` now takes it by value, since the type is `Copy` we don't need the extra layer of indirection. Old tokio API's would often take `&SocketAddr`. async methods still use `<T: ToSocketAddrs>`.
