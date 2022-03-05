---
title: "Many yaks to shave: unprivileged ICMP in tokio"
date: 2022-03-06T22:27:53-05:00
draft: true
---

A yak shave is also an opportunity to learn. I recently ran into some lower level procotols I wasn't familiar with, this turned into an opportunity to learn a deeper layer of tokio I might not have otherwise interacted with. What follows are fresh learnings, so if anything looks wrong contact me so I can correct it.

For context, a while ago I posted about [dhcproto](https://leshow.github.io/post/dhcproto/), go back and have a read to get some DHCP basics if you like. In dhcpv4, when a client wants to get an IP it sends a DISCOVER message, to which the server replies with an OFFER of a potential IP the client could use. A well behaved DHCP client will ARP to see if the IP is in use, while some DHCP servers will send out a ping during this negotiation to see if it's in use. Having IP conflicts on your network is obviously something you want to avoid.

Enter ICMP. A ping uses a protocol called ICMP to communicate with an IP to see if it's in use and if we can get a reply. While it's not perfect, as clients may not respond to pings, it's nice to have the option to ping before an OFFER is sent.

## Yak #1: ICMP what?

I've used the `ping` binary on Linux hundreds of times without so much as a thought to what it was doing. Turns out, it uses [ICMP](https://en.wikipedia.org/wiki/Internet_Control_Message_Protocol) to send an "echo request" and receive an "echo reply" type message. You've likely seen this output before:

```
~
❯ ping 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=58 time=15.1 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=58 time=10.8 ms
64 bytes from 1.1.1.1: icmp_seq=3 ttl=58 time=9.50 ms
64 bytes from 1.1.1.1: icmp_seq=4 ttl=58 time=8.14 ms
^C
--- 1.1.1.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 8.140/10.882/15.137/2.624 ms
```

Now that we know a little about what we're looking at, we can see `icmp_seq` printed out in the pings! My eyes glossed over those fields many times before without so much as a second thought. A digression into the [OSI model](https://en.wikipedia.org/wiki/OSI_model) is outside the scope of this post, but suffice to say: most protocols we interact with live either at the transport layer, like TCP/UDP, or at the application layer, like DNS/HTTP. ICMP lives below transport in layer 3 alongside IP itself. 

As such, ICMP messages will get written directly after the IP header, and an echo request/reply take this form: 

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |     type      |    code       |          checksum             |
 +---------------+---------------+---------------+---------------+
 |         identifier            |           sequence num        |
 +-------------------------------+-------------------------------+
 |                           payload                             |
 +-------------------------------+-------------------------------+
```

The "type" byte says whether it is an echo reply (0) type message or an echo request (8), there are others but they aren't relevant to ping as far as I know. The checksum is computed based on the other data, while the payload and sequence number are arbitrary, they exist to help you match a reply with a request. Oh, that's another thing, since IMCP lives below the transport layer, there are no ports! The contents of those sequence numbers/payloads are necessary to figure out which ping you got a response for. This will come up later.


## Yak #2: How do you send something other than TCP/UDP in tokio?

Okay, so now we know a bit about what ICMP is, how do we send a message with tokio? There are no ICMP sockets in the Rust std lib or tokio, they support TCP/UDP/unix sockets. Essentially, what we need to do is create a raw socket and then build up our ICMP abstraction on top of that, then have that register with the tokio reactor and finally write some async/await methods so we can use it within our larger tokio application.

In C (warning: I don't write a lot of C!), one would use the `socket` syscall [this](https://man7.org/linux/man-pages/man2/socket.2.html) to create an ICMP socket:

```c 
int icmp_soc_fd = socket(AF_INET, SOCK_RAW, IPPROTO_ICMP)
```

In Rust, we've got the lovely [sockets2](https://docs.rs/socket2/latest/socket2/struct.Socket.html) crate that is a thin wrapper over this. In Rust:

```rust 
use sockets2::Socket;

let soc = Socket::new(Domain::IPV4, Type::RAW, Some(Protocol::ICMPV4))
```

What we get is a file descriptor, which we can then use to register with tokio. To do that, we need to remember to set it to `non_blocking` since tokio is an async runtime

```rust 
soc.set_nonblocking(true);
```

But that just gives us a file descriptor that can be polled in a non blocking way, it doesn't give us any async methods to use, tokio needs to know about this file descriptor and it needs to be registered with the tokio reactor so it can know to wake tasks when the socket is ready for data. One way we can do this is by using [`AsyncFd`](https://docs.rs/tokio/latest/tokio/io/unix/struct.AsyncFd.html). This will:
```
/// Creating an AsyncFd registers the file descriptor with the current tokio
/// Reactor, allowing you to directly await the file descriptor being readable
/// or writable. Once registered, the file descriptor remains registered until
/// the AsyncFd is dropped.
```

Perfect! Now we can put that together with `sockets2`

```rust
pub struct Socket {
    socket: AsyncFd<socket2::Socket>,
}

impl Socket {
    pub fn new() -> io::Result<Self> {
        let socket = socket2::Socket::new(Domain::IPV4, Type::RAW, Some(Protocol::ICMPV4))?;
        socket.set_nonblocking(true)?;
        Ok(Self {
            socket: AsyncFd::new(socket)?,
        })
    }

    pub async fn send_to(&self, buf: &[u8], target: &SocketAddr) -> io::Result<usize> {
        let n = loop {
            let mut guard = self.socket.writable().await?;
            match guard.try_io(|inner| inner.get_ref().send_to(buf, &SockAddr::from(*target))) {
                Ok(res) => break res?,
                Err(_would_block) => continue,
            }
        };
        if n != buf.as_ref().len() {
            return Err(io::Error::new(
                io::ErrorKind::Other,
                "failed to send entire packet",
            ));
        }
        Ok(n)
    }

    pub async fn recv(&self, buf: &mut [u8]) -> io::Result<usize> {
        loop {
            let mut guard = self.socket.readable().await?;
            match guard.try_io(|inner| inner.get_ref().read(buf)) {
                Ok(res) => return res,
                Err(_would_block) => continue,
            }
        }
    }
}
```

This is mostly lifted from the example in the tokio docs and slightly tweaked. The idea as far as I can tell is that you poll in a loop to see when the resource is readable or writeable, then attempt your operation with `try_io`. `try_io` seems to do the work, but you'll notice you need to `write` it in a loop, because there is a chance the resource could return a `WouldBlock` and this would mean you need to try writing again at a later time. As far as I know, there is no guarantee you've read all there is to read onto the buffer after that `recv` future finishes.

In trying to understand all this, I found reading the relevant `read` and `write` libc man pages to be interesting. It's nothing new to systems programmers, but as someone who interacts with the underlying libc functions rarely, it's a good read.

## Yak #3: RAW sockets require privileged access (among other things)

Surprise! using the `SOCK_RAW` socket type requires root privileges or the `cap_net_raw` [capability](https://wiki.archlinux.org/title/Capabilities). Why? Because raw sockets allow you to read _all_ of the data on a particular socket. And remember, at this layer there is no concept of "port" so you could potentially see the data coming in on all ports. For some extra pain, the data you get back over the raw socket hasn't even had its IP header decoded, so when you're parsing ICMP on the reply, you must also parse the IPv4 header!

But hold on, we ran `ping` without sudo and it worked just fine, why? Well, it may have setuid permissions on older linux systems, on newer ones it may have the `cap_net_raw` capability. OR apparently, on even newer kernels like mine

```
~
❯ getcap /bin/ping
```

There are no capabilities required at all? This led me to find this [patch](https://lwn.net/Articles/420800/). I gather that implementing ICMP echo request/reply is fairly common, so kernel devs have given us a mechanism to not have to use the raw socket type and still send ICMP messages! Remember the `socket` syscall?

```c
  socket(PF_INET, SOCK_DGRAM, IPPROTO_ICMP)
```

If the type is set to `SOCK_DGRAM` instead of `SOCK_RAW`, it can be created without special privileges. In this case, you don't need capabilities, or root, or setuid, and you don't need to decode the IPv4 header on the reply... bonus!

The kernel will make sure that you can only send ICMP packets with a header type of request or reply, I believe. In the time from this patch, ICMPV6 support has been added for `SOCK_DGRAM`.

## Yak #4: ICMP Identifiers with DGRAM sockets

Remember how I said the lack of ports would come up later? Well when you use a raw socket, the identifier you write to the socket will be what you get in your response. Using that identifier & sequence number & perhaps payload, you can match up your requests to replies. However, if you use a DGRAM socket, this is not the case:

```
Message identifiers (octets 4-5 of ICMP header) are interpreted as local
ports. Addresses are stored in struct sockaddr_in. No port numbers are
reserved for privileged processes, port 0 is reserved for API ("let the
kernel pick a free number"). There is no notion of remote ports, remote
port numbers provided by the user (e.g. in connect()) are ignored.
```

What this ended up manifesting itself as in my case was that the random u16 identifier I was generating and writing to the socket was immediately being overwritten by the kernel into something else that more resembled a session id. As a result, I could no longer use it to help match requests to replies on the socket. The end result of this is that I just use the sequence number and the payload as the identifying fields. ICMP replies must contain the same payload as what they received, so there's an arbitrary width block of bytes you can use to match up your requests/replies.

## Conclusion

With a underlying socket type that has async/await methods to send & recv byte buffers, a higher level abstraction that speaks ICMP is fairly easy. I was originally manually encoding/decoding the message (after all it's only a few bytes) but ended up offloading that to `pnet_packet` which has methods for computing the checksum too. This lets some rather fiddly code get deleted. The `pnet` family of crates has come in handy more times than I can count.

I'm writing this half to share what I've learned and half so I can read the blog later and remember what I've forgotten. If you made it this far, thank you!

