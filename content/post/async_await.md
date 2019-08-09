---
title: "Updating to Async/Await"
date: 2019-08-09T13:52:08-04:00
draft: true
---

Unless you've been living under a rock; you know async/await is coming to rust stable. My last post was about implementing a simple protocol using manual futures, and interacting with [tokio](https://github.com/tokio-rs/). It's only fitting, then, that I update the lib that post was inspired by to async/await and report back on my findings. If you're curious about my library or you use the window manager i3, it's available [here](https://github.com/leshow/tokio-i3ipc/) or on crates under [tokio-i3ipc](https://crates.io/crates/tokio-i3ipc). The version discussed here is currently unpublished, I'm waiting for the syntax to be released on stable and tokio to push their new version.

## Background

Suffice to say, much has been said about the foundations of the async ecosystem. The high level overview is that there is a `Future` trait that provides the method `poll` that returns the enum `Poll::Ready`/`Poll::Pending`. Just like how synchronous io in Rust is built on `io::Read`/`io::Write`, the asynchronous world is built on `AsyncRead`/`AsyncWrite`, each of which have methods like `poll_read` or `poll_write` (respectively) that return a type which can be `poll`ed. That probably didn't make any sense unless you were already familiar with the ecosystem, so I'm assuming some previous knowledge here (and completely skipping `Pin`)!

## Protocol refresher

The protocol is i3's IPC. It is text-based and communicates over a unix socket. The basic format is:

```
"i3-ipc"<payload len (u32)><message type (u32)><json encoded payload>
```

## Updating hand implemented futures

I originally thought I would just take some of the manual futures I had implemented, update them to the new `std::futures` format and be done with it. Doing something like that would turn this (0.1 futures):

```rust
use futures::{try_ready, Async, Future, Poll};
use serde::de::DeserializeOwned;
use std::{io as stio, marker::PhantomData};
use tokio::io::{self as tio, AsyncRead};

impl<D, S> Future for I3Msg<D, S>
where
    S: AsyncRead,
    D: DeserializeOwned,
{
    type Item = MsgResponse<D>;
    type Error = stio::Error;
    fn poll(&mut self) -> Poll<Self::Item, Self::Error> {
        let mut buf = [0_u8; 14];

        let (rd, init) = try_ready!(tio::read_exact(&mut self.stream, &mut buf).poll());

        if &init[0..6] != MAGIC.as_bytes() {
            panic!("Magic str not received");
        }
        let payload_len = u32::from_ne_bytes([init[6], init[7], init[8], init[9]]) as usize;
        let msg_type = u32::from_ne_bytes([init[10], init[11], init[12], init[13]]);
        let mut buf = vec![0_u8; payload_len];
        let (_rdr, payload) = try_ready!(tio::read_exact(rd, &mut buf).poll());

        Ok(Async::Ready(MsgResponse {
            msg_type: msg_type.into(),
            body: serde_json::from_slice(&payload[..])?,
        }))
    }
}
```

Using a more or less direct translation would lead to (futures-preview):

```rust
use futures::{future::Future, ready, Poll};
use serde::de::DeserializeOwned;
use std::{io as stio, marker::PhantomData, pin::Pin, task::Context};
use tokio::io::{self as tio, AsyncRead, AsyncReadExt};

impl<D, S> Future for I3Msg<D, S>
where
    S: AsyncRead + AsyncReadExt + Unpin,
    D: DeserializeOwned,
{
    type Output = stio::Result<MsgResponse<D>>;

    fn poll(mut self: Pin<&mut Self>, ctx: &mut Context<'_>) -> Poll<Self::Output> {
        let mut init = [0_u8; 14];

        let len_read = ready!(&mut self.stream.read_exact(&mut init).poll(ctx));

        if &init[0..6] != MAGIC.as_bytes() {
            panic!("Magic str not received");
        }
        let payload_len = u32::from_ne_bytes([init[6], init[7], init[8], init[9]]) as usize;
        let msg_type = u32::from_ne_bytes([init[10], init[11], init[12], init[13]]);
        let mut buf = vec![0_u8; payload_len];
        let payload_len = ready!(self.stream.read_exact(&mut buf).poll(ctx));

        Poll::Ready(Ok(MsgResponse {
            msg_type: msg_type.into(),
            body: serde_json::from_slice(&buf[..])?,
        }))
    }
}
```

There's a couple important things to note here. `Future::poll` no longer has associated types `Item` and `Error`. Instead, there is now a single type `Output` that if you want to represent something that can fail, should return a `Result`.

The `Poll` type synonym has changed; gone is the `Async` type, it's been replaced by the enum `Poll`, with variants `Ready` or `Pending`.

You may have noticed that `read_exact` when polled, no longer returns a tuple containing a `AsyncRead`er and a buffer that was written to. In futures 0.1, you had to do this dance with things that implemented `AsyncRead`/`AsyncWrite` and effectively thread instances of the stream through your application. You don't seem to have to do that anymore, you can call `read_exact` on `stream` itself, and you don't need to get back a new reference to it.

I abandonded this path pretty quickly. For one, it wasn't bearing much fruit. It's a little nicer, but writing futures is still sisyphean. Perhaps most importantly, it didn't improve the API of the end product very much. In trying to understand some of the changes to the `Future` trait, I spent some time reading the stdlib definitions for [Pin](https://doc.rust-lang.org/std/pin/index.html), and [Unpin](https://doc.rust-lang.org/std/marker/trait.Unpin.html). The `Pin` docs are very good and I recommend re-reading them a few times and experimenting with `Pin`'ing something. Try to make a self-referencing type like they describe in the docs.

## Embracing the syntax

I have a synchronous version of this protocol, and after playing around with async/await for a little bit, I realized I could just copy my sync API and add the `async` keyword and pretty much call it a day.

I deleted all my manually implemented futures and the above turned into:

```rust
#[derive(Debug)]
pub struct I3 {
    stream: UnixStream,
}

impl I3 {
    async fn _decode_msg(&mut self) -> io::Result<(u32, Vec<u8>)> {
        let mut init = [0_u8; 14];
        let _len = self.stream.read_exact(&mut init).await?;

        if &init[0..6] != MAGIC.as_bytes() {
            panic!("Magic str not received");
        }
        let payload_len = u32::from_ne_bytes([init[6], init[7], init[8], init[9]]) as usize;
        let msg_type = u32::from_ne_bytes([init[10], init[11], init[12], init[13]]);

        let mut payload = vec![0_u8; payload_len];
        let _len_read = self.stream.read_exact(&mut payload).await?;

        Ok((msg_type, payload))
    }

    pub async fn read_msg<D>(&mut self) -> io::Result<MsgResponse<D>>
    where
        D: DeserializeOwned,
    {
        let (msg_type, payload) = self._decode_msg().await?;
        Ok(MsgResponse {
            msg_type: msg_type.into(),
            body: serde_json::from_slice(&payload[..])?,
        })
    }
}
```

I think it's a huge improvement, both in ease of implementation and the end result of the API it enables. It's a little less generic, but only because I chose to not parameterize `UnixStream` above.

That brings me to what I think is the salient thing about this new syntax: it enables a synchronous looking API that is a pleasure to read and write. It gets rid of a lot of the sharp edges in writing futures code (threading streams through combinators, dealing with `Poll`, and `try_ready`, etc). Error handling is also, so, so much nicer this time around.

## Streams

I haven't had to manually implement the `Stream` trait yet, but I do include a codec using tokio's `Decoder` trait which enables you to listen to a stream of i3 events from a `UnixStream`. I haven't had to modify the code for this at all since pointing to tokio's master branch. Using it was a bit hairy before:

```rust
// code removed
let framed = FramedRead::new(stream, EventCodec);
let sender = framed
    .for_each(move |evt| {
        // here tx is a futures::mpsc::channel
        let tx = tx.clone();
        tx.send(evt)
            .map(|_| ())
            .map_err(|e| io::Error::new(io::ErrorKind::BrokenPipe, e))
    })
    .map_err(|err| println!("{}", err));
tokio::spawn(sender);
// code removed
```

tokio's `FramedRead` implements the `Stream` trait, so really all I needed to do was add a method that returned it from my type:

```rust
pub fn listen(self) -> FramedRead<UnixStream, codec::EventCodec> {
    FramedRead::new(self.stream, codec::EventCodec)
}
```

You can use `Stream`s in a `while let` loop

```rust
#[tokio::main]
async fn main() -> io::Result<()> {
    let mut i3 = I3::connect().await?;
    let _resp = i3.subscribe([Subscribe::Window]).await?;

    let mut listener = i3.listen();
    while let Some(event) = listener.next().await {
        println!("{:#?}", event?);
    }
    Ok(())
}
```

The implicit runtimes enabled by proc macros are really a boon to readability.

## Conclusion

I'm a big fan of the new syntax, if you couldn't tell already. Looking from the outside, they took something that was difficult and made it more accessible, which is great, because that's pretty much Rust's mission statement. It's cool to see a confluence of these disparate language features meeting to solve a set of problems. When I look back on the things that have been added to the lang in the last few years: `impl Trait`, proc macros, and now `async`/`await` I see a worthy addition to the language that's already paying dividends.
