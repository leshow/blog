---
title: "Introducing: DHCProto"
date: 2021-09-14T14:08:51-04:00
draft: false
---

[`DHCProto`](https://crates.io/crates/dhcproto) is available now on [crates.io](https://crates.io/crates/dhcproto). [`DHCProto`](https://crates.io/crates/dhcproto) implements a parser/encoder for DHCPv4 & DHCPv6. This crate was born out of the dearth of DHCP offerings currently available. It has a comprehensive list of option types implemented, includes serializing a message back to bytes, and DHCPv6. This library was written at my work ([Bluecat](https://bluecatnetworks.com/)), and they have graciously let us open source it, available on [github](https://github.com/bluecatengineering/dhcproto) for issues/comments/PRs. The encoder/decoder traits are inspired a lot by `trust-dns-proto`.

DHCP is perhaps not the most popular protocol, but never-the-less integral to networks everywhere. We hope to improve the situation marginally by having a high-quality (I hope) library released for everyone to use/contribute to.

**Interested?**: If you know Rust and have an interest/experience in writing network services, we are always looking for talented people so feel free to reach out to me!

## A brief and incomplete history of DHCP

DHCPv4 was based on the anachronistic [BOOTP](https://en.wikipedia.org/wiki/Bootstrap_Protocol), which was itself created to solve problems with [RARP](https://en.wikipedia.org/wiki/Reverse_Address_Resolution_Protocol). BOOTP was defined in RFC 951 and run from floppy disks. The relatively low RFC number should give another indication to its age. This protocol was then evolved to support dynamic addresses, which is where DHCP (v4) comes into the picture.

DHCP was first introduced in RFC 1531 in '93 (1), later finalized in '97. DHCP takes the same header layout as BOOTP:

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

But extends it by adding more `options`. This is a variable-width section of the protocol that contains potentially many different things, but most importantly a new option type was added called 'message type'. The message types are as follows:

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

A good mnemonic to remember the 'happy-path' for DHCP address assignment is "DORA", as in Discover, Offer, Request, Ack. The discover/request parts are sent by the client and the other two by the server in response. The free "TCP/IP Guide" has a visualization of the state machine of DHCP [here](http://www.tcpipguide.com/free/t_DHCPGeneralOperationandClientFiniteStateMachine.htm). It's worth mentioning, because client addresses are unknown at the start of this process, many DHCP messages are transmitted over UDP broadcast.

## Enter DHCPv6

While some protocols like DNS have responded to IPv6 by adding additional record types and extending the protocol, this was not possible with DHCP. The protocol has some fundamental assumptions baked in about IPv4, as best I can tell. Firstly, the address widths in the header are too small for IPv6, which requires 128 bits instead of 32. Secondly, there is no IPv6 broadcast for discovery. Given these, and probably other concerns, a new protocol was created. Here is the header format:

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

The message type is now first thing, instead of being stuffed in the variable with `options` section, followed by a transaction ID. What follows is variable width options, which we won't get into here. But suffice to say the header format is much simpler, and every message must now have a message type.

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

As you can see the message types here are totally different compared to v4, but fear not! There is a new mnemonic to remember: "SARR". This is again the happy path: Solicit, Advertise, Request, Reply. Since IPv6 does not support UDP broadcast, DHCPv6 uses a dedicated multicast address. There are more complicated things in the protocol like SLAAC (stateless address auto configuration) but this is well beyond the scope of this post.

Sources:

[1](https://www.isc.org/dhcphistory/)
[2](https://datatracker.ietf.org/doc/html/rfc2132#section-9.6)
[3](http://www.networksorcery.com/enp/protocol/dhcpv6.htm)
[4](https://datatracker.ietf.org/doc/html/rfc8415#section-8)
