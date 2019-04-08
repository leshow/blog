---
title: "Protocols in Tokio (i3 IPC)"
date: 2019-04-07T18:53:18-04:00
draft: false
---

There's a dearth of blog posts online that cover the details of implementing a custom protocol in tokio, at least that I've found. I'm going to cover some of the steps I went through in implementing an async version i3wm's [IPC](https://i3wm.org/docs/ipc.html). Granted, I've not finished my library to a point I'm comfortable releasing it, but I hope I can provide some examples for the aspiring async IO enthusiast that I wish I had when I started. A basic knowledge of `futures` and `tokio` will be helpful.

## The Protocol

i3's protocol is pretty simple. It uses a unix socket, and all messages sent to and from i3 follow the same general pattern:

```
"i3-ipc"<payload len (u32)><message type (u32)><json encoded payload>
```

We can subscribe to events like "window" and "workspace", which will have data written to the socket when you do things like change active windows or switch workspaces. These strings are json encoded in an array and sent as the payload along with the 'subscribe command'. In response, we get a payload of `{"success": true}` (or false).

Without further ado, let's get to the implementation.

## Using combinators and tokio::io

I think the most obvious place to start is using the built in combinators that come with `futures` and `tokio`. With the advent of Rust 1.26 and impl Trait we can return this as a `impl Future<Item=(), Error=()>`.

While tokio has some good examples using [TcpListener](https://tokio.rs/docs/io/overview/) for processing incoming tcp connections, I couldn't find much for bidirectional communication using `UnixStream`. The first challenge I found, being naive of the tokio ecosystem, is that `tokio_uds::UnixStream` returns a `ConnectFuture`. Rather than actually connecting to anything this is a type which implements Future (rather obvious in hindsight) that must be polled in order to actually connect to the socket.

Therefore, we must use `and_then` to access the `UnixStream` that will be resolved after the `ConnectFuture` runs:

```rust
use tokio_uds::UnixStream;

let fut = UnixStream::connect(path).and_then(|stream: UnixStream| {
    // do wonderful things
});

// and of course, to actually run it all
tokio::run(fut);
```

So, the back and forth looks something like this:

```rust
fn subscribe(events: Vec<Event>) -> io::Result<()> {
    let fut = UnixStream::connect(socket_path()?)
        .and_then(move |stream| {
            let buf = subscribe_payload(events);
            tokio::io::write_all(stream, buf)
        })
        .and_then(|(stream, _buf)| {
            decode_response(stream, |msg_type: u32, buf: Vec<u8>| {
                let s = String::from_utf8(buf.to_vec()).unwrap();
                println!("{:?}", s); // {success:true}
                dbg!(msg_type); // 2 (for subscribe)
            })
        })
        .and_then(|stream| {
            decode_response(stream, |evt_type: u32, buf: Vec<u8>| {
                let resp = decode_evt(evt_type, buf); // mostly does serde deserializing, not shown
                dbg!(&resp);
            })
        })
        .map(|_| ())
        .map_err(|e| eprintln!("{:?}", e));

    tokio::run(fut);
    Ok(())
}
```

Here, we are resolving that `connect`, then writing the subscribe data to the Stream using `io::write_all`. After that's done we must read from the Stream in decode_response (the `{success:true}` response). And only then will we wait to receive the first event, and decode.

`decode_response` is a Future itself. It takes a closure that receives a message type and buffer and presumably does something with the data:

```rust
fn decode_response<F>(stream: UnixStream, f: F) -> impl Future<Item = UnixStream, Error = io::Error>
where
    F: Fn(u32, Vec<u8>),
{
    let buf = [0; 14];
    tokio::io::read_exact(stream, buf).and_then(|(stream, initial)| {
        if &initial[0..6] != MAGIC.as_bytes() {
            panic!("Magic str not received");
        }
        let payload_len = LittleEndian::read_u32(&initial[6..10]) as usize;
        let evt_type = LittleEndian::read_u32(&initial[10..14]);

        let buf = vec![0; payload_len];
        tokio::io::read_exact(stream, buf).and_then(move |(stream, buf)| {
            f(evt_type, buf); // do something
            future::ok(stream)
        })
    })
}
```

We don't know the size of the message before we start reading it, but we do know that the first bit, `i3-ipc<payload len><msg type>` is 14 bytes. 6 for `i3-ipc` and 4 each for the two u32's. What follows after that is the payload response and it could be any size. So we are delineating between the two parts of the message; the part of known size and the part with unknown size. We finish by returning the stream to potentially be used again.

The major issue with this, besides basically not handling errors, is it only reads a single event. We want to listen to a stream of window change events. And if you think about it, that makes sense, we're returning a `Future` for `decode_response` when what we need to be doing is working with a `Stream`.

## Codecs

I was stumped on this part. But a few options were available, and I think these are (broadly) the solution to writing tokio code in general:

- (1) Use combinators and abstract using functions and `impl Future`
- (2) Implement custom `Future` and/or `Stream` types
- (3 - perhaps more specific to this problem) use `Encoder`/`Decoder` and write a codec

I began reading some more tokio documentation and it seemed more and more like the thing I wanted to do was create a [framed stream](https://tokio.rs/docs/going-deeper/frames/). The example in the tokio docs is writing a protocol that splits input into frames by the newline delimiter `\n`. The i3 protocol is slightly more complex but not by a lot.

As far as I can tell option 1 doesn't really work for `Stream`. By that I mean, it doesn't look like it's possible to return an `impl Stream` of i3 event responses. The path is to implement a `Future` to decode a single message, then build a `Stream` out of those futures which naturally leads us to option 2.

I had a suspicion that by going through all that I'd be re-building machinery that `Encoder` / `Decoder` do anyway. I did a GitHub search for `tokio_codec` and limited the search to Rust to try and find some examples. Most codecs appear to be unit structs, mine was no different:

```rust
struct EventCodec;
```

After the initial success response from sending subscribe all our stream needs to do is decode messages and run the json parts through serde, therefore we only need `Decoder` for `EventCodec`.

```rust
impl Decoder for EventCodec {
    type Item = event::Event;
    type Error = io::Error;
    fn decode(&mut self, src: &mut BytesMut) -> Result<Option<Self::Item>, io::Error> {
        if src.len() > 14 {
            if &src[0..6] != MAGIC.as_bytes() {
                return Err(io::Error::new(
                    io::ErrorKind::Other,
                    format!("Expected 'i3-ipc' but received: {:?}", &src[0..6]),
                ));
            }
            let payload_len = LittleEndian::read_u32(&src[6..10]) as usize;
            let evt_type = LittleEndian::read_u32(&src[10..14]);
            if src.len() < 14 + payload_len {
                Ok(None)
            } else {
                let evt = decode_event(evt_type, src[14..].as_mut().to_vec())?;
                src.clear(); // !!
                Ok(Some(evt))
            }
        } else {
            Ok(None)
        }
    }
}
```

Our `Decoder` produces `Event` responses, which is the deserialized result of parsing some IPC payload, or an `Error`. Returning `Ok(None)` from `decode` signifies to tokio to continue consuming data. I'm not sure if the length checking is necessary since in practice it seems like it only ever does a single read operation for the entire response (at least in my tests), but it seems right for a function that can say "hey, I need more data".

The first time I tried using this, it accepted a single event but decoded it infinitely. My console filled with the exact same event being processed again and again. After some stackoverflow help I added `src.clear()`, to clear the buffer after successfully decoding the event and produce a single frame per event.

You can use Decoders and Encoders to turn a `UnixStream` or `TcpStream` into frames using:

```rust
let framed = FramedRead::new(stream, EvtCodec);
let sender = framed
    .for_each(move |evt: event::Event| {
        // do something
    })
    .map_err(|err| println!("{}", err));
tokio::spawn(sender);
```

I connected this with the original `subscribe` function to produce:

```rust
pub fn subscribe(
    rt: tokio::runtime::current_thread::Handle,
    tx: Sender<event::Event>,
    events: Vec<Subscribe>,
) -> io::Result<()> {
    let fut = UnixStream::connect(socket_path()?)
        .and_then(move |stream| {
            let buf = subscribe_payload(events);
            tokio::io::write_all(stream, buf)
        })
        .and_then(|(stream, _buf)| {
            decode_response(stream, |msg_type: u32, buf: Vec<u8>| {
                let s = String::from_utf8(buf.to_vec()).unwrap();
                println!("{:?}", s); // {success:true}
                dbg!(msg_type); // 2 (for subscribe)
            })
        })
        .and_then(move |stream| {
            let framed = FramedRead::new(stream, EvtCodec);
            let sender = framed
                .for_each(move |evt| {
                    let tx = tx.clone();
                    tx.send(evt)
                        .map(|_| ())
                        .map_err(|e| io::Error::new(io::ErrorKind::BrokenPipe, e))
                })
                .map_err(|err| println!("{}", err));
            tokio::spawn(sender); // !
            Ok(())
        })
        .map(|_| ())
        .map_err(|e| eprintln!("{:?}", e));

    tokio::spawn(fut);
    Ok(())
}
```

I decided to pass in a `Sender` and use a `futures::mpsc::channel` to communicate the events received over the socket to elsewhere. I think this is a nice approach and allows this function (after it's been spruced up) to potentially be exported as a library function and have users pass the Sender in and listen on the other side of the channel for responses.

There's an extra `spawn` for the bit that runs `sender`. This is because `sender` is still a Future. If we return it instead of spawning it, then `for_each` would wait for it to complete before it accepts the next response. That's probably not a big deal here since i3 will probably only send a single event at a time, but there's not much point in doing all this work if we don't enable ourselves to actually use the concurrency provided.

## Conclusion

I started this tokio adventure feeling very much like I was in over my head. However the more time I spent interacting with the various bits of the ecosystem the more I realized the parts that seemed obscure and magic were very much non-magical. The Future and Stream traits, along with the tokio ecosystem built on top of it are very well thought out and while difficult initially, are pretty damn cool and not so bad after you spend some time with them.

I'm not quite done building this library and polishing things off, I will write a part 2 after everything is completed. I am by no means a tokio expert so if anyone catches any errors or has some tips, I'd love to hear the feedback. Otherwise I hope this is helpful to someone. 'Till next time

Cheers
