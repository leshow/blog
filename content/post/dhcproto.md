---
title: "dhcproto"
date: 2021-09-14T14:08:51-04:00
draft: true
---

Announcing a new Rust library! [`dhcproto`](https://crates.io/crates/dhcproto) is available now on [crates.io](https://crates.io/crates/dhcproto). [`dhcproto`](https://crates.io/crates/dhcproto) aims to be a full DHCPv4 & DHCPv6 decoder/encoder. There are not many DHCP crates out there with a comprehensive list of option types implemented, and none had everything we needed particularly when it came to serializing back to bytes and dhcpv6. This library was written at my work ([Bluecat](https://bluecatnetworks.com/)), and they have graciously let us open source it, available on [github](https://github.com/bluecatengineering/dhcproto) for issues/comments/PRs.

DHCP is perhaps not the most popular protocol, but never-the-less integral to networks everywhere. We hope to improve the situation marginally by having a high-quality (I hope) library released for everyone to use/contribute to.

**Note**: If you know Rust and are a network aficionado, we are always looking for people so feel free to reach out to me.

## a brief (and mostly wrong) account of DHCP

DHCPv4 was based on the anachronistic [BOOTP](https://en.wikipedia.org/wiki/Bootstrap_Protocol), originally defined in RFC 951 and requiring a floppy disk. The relative low RFC number should give another indication to its age. This protocol was then extended to support dynamic addresses, which is where DHPCv4 comes into the picture.

DHCPv4 was first introduced in RFC 1531 in '93 (1), later finalized in '97. DHCP takes the same header layout as BOOTP:

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |     op (1)    |   htype (1)   |   hlen (1)    |   hops (1)    |
 +---------------+---------------+---------------+---------------+
 |                            xid (4)                            |
 +-------------------------------+-------------------------------+
 |           secs (2)            |           flags (2)           |
 +-------------------------------+-------------------------------+
 |                          ciaddr  (4)                          |
 +---------------------------------------------------------------+
 |                          yiaddr  (4)                          |
 +---------------------------------------------------------------+
 |                          siaddr  (4)                          |
 +---------------------------------------------------------------+
 |                          giaddr  (4)                          |
 +---------------------------------------------------------------+
 |                          chaddr  (16)                         |
 +---------------------------------------------------------------+
 |                          sname   (64)                         |
 +---------------------------------------------------------------+
 |                          file    (128)                        |
 +---------------------------------------------------------------+
 |                          options (variable)                   |
 +---------------------------------------------------------------+
```

But extends it by adding more `options`. This is a variable-width section of the protocol that contains many different things, but most importantly contains a new option called 'message type'. The message types are as follows:

```text
 Value   Message Type
 -----   ------------
   1     DHCPDISCOVER
   2     DHCPOFFER
   3     DHCPREQUEST
   4     DHCPDECLINE
   5     DHCPACK
   6     DHCPNAK
   7     DHCPRELEASE
   8     DHCPINFORM
```

(2)

A good mnemonic to remember the 'happy-path' for DHCP address assignment is "DORA", as in Discover, Offer, Request, Ack. The free "TCP/IP Guide" has a wonderful visualization of the state machine of DHCP [here](http://www.tcpipguide.com/free/t_DHCPGeneralOperationandClientFiniteStateMachine.htm). It's worth mentioning, because client addresses are unknown at the start of this process, many of the messages are transmitted over UDP broadcast.

## Enter DHCPv6

I have worked a lot with DNS, and the DNS response to IPv6 has been to extend the existing protocol to with relevant bits to make everything work (excuse my horrible summation). DHCP did not choose the same path. The protocol has some fundamental assumptions baked in about IPv4. Firstly, the address widths in the header are too small for IPv6, which requires 128 bits instead of 32. Secondly, there is no IPv6 broadcast. As such, it seems like a rethink of DHCP was in order.

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    msg-type   |               transaction-id                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
.                            options                            .
.                 (variable number and length)                  .
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

(4)

The message type is the first thing, followed by a transaction ID (though in the authors humble opinion at an oddly sized 3 bytes). What follows is variable width options, which we won't get into here.

The `msg-type` field can have the following possible values:

```text
Value    Message Type
-----    ------------
  1        SOLICIT.
  2        ADVERTISE.
  3        REQUEST.
  4        CONFIRM.
  5        RENEW.
  6        REBIND.
  7        REPLY.
  8        RELEASE.
  9        DECLINE.
  10       RECONFIGURE.
  11       INFORMATION-REQUEST.
  12       RELAY-FORW.
  13       RELAY-REPL.
  14       LEASEQUERY.
  15       LEASEQUERY-REPLY.
  16       LEASEQUERY-DONE.
  17       LEASEQUERY-DATA.
```

(3)

As you can see the message types here are totally different, but fear not! There is another mnemonic to remember: "SARR". This is again the happy path of address assignment, Solicit, Advertise, Request, Reply. To address the "no broadcast" restriction in IPv6, DHCPv6 uses a dedicated multicast address. There are more complicated things added to the protocol like SLAAC (stateless address auto configuration) but this is well beyond the scope of this post.

Sources:

- [1](https://www.isc.org/dhcphistory/)
- [2](https://datatracker.ietf.org/doc/html/rfc2132#section-9.6)
- [3](http://www.networksorcery.com/enp/protocol/dhcpv6.htm)
- [4](https://datatracker.ietf.org/doc/html/rfc8415#section-8)
