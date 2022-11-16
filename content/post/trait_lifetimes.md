---
title: "Deciding Trait Lifetimes"
date: 2022-06-26T00:18:24-04:00
draft: true
---

If you've read any of this blog recently, you know that I've been working on DHCP in Rust at Bluecat. We have open sourced some of our work so far, of note is the [dhcproto](https://github.com/bluecatengineering/dhcproto) crate. You can read about it [here](https://leshow.github.io/post/dhcproto/) but I'll summarize briefly:

- It's a DHCPv4/v6 encoder/decoder
- Aims to support as many options with first-class types as possible
- It borrows ideas from trust-dns-proto

## trust-dns-proto

`trust-dns-proto` has two main traits, `BinDecodable` and `BinEncodable`

```
pub trait BinDecodable<'r>: Sized {
    fn read(decoder: &mut BinDecoder<'r>) -> ProtoResult<Self>;

    fn from_bytes(bytes: &'r [u8]) -> ProtoResult<Self> { ... }
}
```

```
pub trait BinEncodable {
    fn emit(&self, encoder: &mut BinEncoder<'_>) -> ProtoResult<()>;

    fn to_bytes(&self) -> ProtoResult<Vec<u8>> { ... }
}
```

`BinDecodable` is "for types which are serializable to and from DNS formats"
`BinEncodable` is "a type which can be encoded in DNS binary format"

In other words, you decode to turn bytes pulled off the wire into a usable type, and you encode to turn that type back into bytes that you can put on the wire. This makes a lot of sense to me, so when I was writing dhcproto, I thought of it as the `trust-dns-proto` of DHCP (aspirationally). I couldn't use the traits directly because the error type and a few other things didn't make sense for dhcproto so I "borrowed" ideas.

## dhcproto

Our traits are very similar:

```
pub trait Decodable: Sized {
    fn decode(decoder: &mut Decoder<'_>) -> DecodeResult<Self>;

    fn from_bytes(bytes: &[u8]) -> DecodeResult<Self> { ... }
}
```

```
pub trait Encodable {
    fn encode(&self, e: &mut Encoder<'_>) -> EncodeResult<()>;

    fn to_vec(&self) -> EncodeResult<Vec<u8>> { ... }
}
```

In fact, the `Encodable` trait is basically copypasta. However, there is one change in `Decodable`; the lifetime is no longer declared in the trait itself, it's polymorphic over the `decode` method. That anonymous lifetime expands into:

```
fn decode<'a>(decoder: &mut Decoder<'a>) -> DecodeResult<Self>
```

I chose this because none of the types produced by decode contained any references when I started, i.e. there were no references in `Self`, only in the `decoder`. Both of these traits are used to implement dhcpv4 & dhcpv6, both of which have fundamentally different on the wire structures and types. I often try to be generic when I can, for instance, if you're defining a type that will be produced by a stream of UDP frames, you end up with something like:

```
struct Msg<T> {
	msg: T
	src: SocketAddr,
	dst: SocketAddr,
	... other stuff
}
impl<T: Decodable + Encodable> Msg<T> {
	fn new(bytes: Bytes) -> Result<Self> {
		let mut dec = Decoder::new(&buf);
        Ok(Self { msg: T::decode(&mut dec)? })
	}
}
```

This way the `Msg` type can be used for v4 & v6, if we need method specifically for v4, it's easy enough to `impl Msg<v4::Message>`, but invariably lots of functions will be similar.

## Conflict!

So, all was right with the world, my types & traits were working fine. Suddenly, a new RFC appears (well, new to me) [RFC 3396](https://datatracker.ietf.org/doc/html/rfc3396). You see, up until this point I had assumed the variable width options section for DHCPv4 followed a few rules:

- That there was only one option of each type, no duplicates
- That the max length of an option was 255 bytes

The general form of an option from the original DHCP RFC is

(Option 1 as an example)
Code Len Subnet Mask
+-----+-----+-----+-----+-----+-----+
| 1 | 4 | m1 | m2 | m3 | m4 |
+-----+-----+-----+-----+-----+-----+

The first byte is the option code, the next is the length, and then you read the length to get the data in the option and decode it depending on what it is. RFC 3396 changes that to allow for options to be longer than 255 bytes. You can include multiple of the same option code in the buffer consecutively, and the decoder is meant to concatenate the data portions.

Ex.
Code Len Data Code Len Data
+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+
| x | n | m1 | ... | x | i | mn+1| ... | mn+i|
+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+

Here, if both `x` opt codes are the same, you would have a single option of length `n + i`, by concatenating the data sections together.

So, now what was a simple decode process becomes a little trickier. You need to hold onto the last option and if the next option has the same code, extend the data portion of the first. The simple solution to this is just to allocate an intermediate `Vec<u8>` for each option during decode and concat data if needed, if the next option code is different then proceed with decoding.

In all likelihood, this would have been fine, but it _really_ bothered me that 99% of the options decoded would be less than 255 bytes, making the allocation was unnecessary most of the time. My solution was to use the wonderful `Cow` type from the std lib. If you're not familiar:

`std::borrow::Cow`:

```
pub enum Cow<'a, B> where
    B: 'a + ToOwned + ?Sized,  {
    Borrowed(&'a B),
    Owned(<B as ToOwned>::Owned),
}
```

Cow is a "clone-on-write" type. It looks complicated but essentially, it is a type that's either a reference or an "owned" value, in this case `Cow<'a, [u8]>`. Either a `&[u8]` slice or a `Vec<u8>`. The type of the owned variant is determined through that `<B as ToOwned>::Owned` bound. When benchmarked, this beat the naive always allocate version by a good margin. And although maybe not "zero cost" as there is still an intermediate, albeit mostly stack allocated struct, performance was 5-10% worse on a process that's under 1000ns and so would be imperceptible.

## Here's the rub

This brings me back to the trait definitions. I had declared the trait without a lifetime, so implementing something like:

```
// our intermediate type
struct Opt<'a> {
	code: u8,
	buf: Cow<'a, [u8]>
}
impl<'a> Decodable for Opt<'a> {
    fn decode<'b>(dec: &mut Decoder<'b>) -> DecodeResult<Self> {
        let [code, len] = dec.peek::<2>()?;
        let buf = Cow::from(dec.read_slice(len as usize + 2)?);
        Ok(Opt { code, buf })
    }
}
```

error!

```
error[E0495]: cannot infer an appropriate lifetime for lifetime parameter `'a` due to conflicting requirements
  --> src/lib.rs:21:33
   |
21 |         let buf = Cow::from(dec.read_slice(len as usize + 2)?);
   |                                 ^^^^^^^^^^
   |
note: first, the lifetime cannot outlive the lifetime `'b` as defined here...
  --> src/lib.rs:19:15
   |
19 |     fn decode<'b>(dec: &mut Decoder<'b>) -> DecodeResult<Self> {
   |               ^^
note: ...so that the types are compatible
  --> src/lib.rs:21:33
   |
21 |         let buf = Cow::from(dec.read_slice(len as usize + 2)?);
   |                                 ^^^^^^^^^^
   = note: expected `&mut Decoder<'_>`
              found `&mut Decoder<'b>`
note: but, the lifetime must be valid for the lifetime `'a` as defined here...
  --> src/lib.rs:18:6
   |
18 | impl<'a> Decodable for Opt<'a> {
   |      ^^
note: ...so that the types are compatible
  --> src/lib.rs:22:9
   |
22 |         Ok(Opt { code, buf })
   |         ^^^^^^^^^^^^^^^^^^^^^
   = note: expected `Result<Opt<'a>, _>`
              found `Result<Opt<'_>, _>`
```

Duh! I thought, of course `trust-dns-proto` got it right in the first place. Putting a lifetime in the trait allows it to be implemented for types containing references. So let's just change that!

```
pub trait Decodable<'a>: Sized {
    fn decode(decoder: &mut Decoder<'a>) -> DecodeResult<Self>;

    fn from_bytes(bytes: &'a [u8]) -> DecodeResult<Self> { ... }
}
```

Great, now I can implement for `Opt<'a>`. It's a breaking change and I had to update a few more places... but this library is all pre-1.0 anyway. So fine.

## My surprise

I thought that was the end of it. I merged long form encoding into master with the lifetime change in the trait (but not yet released) and went about my day. It was later, when I went into my projects that actually put this code to use in a real project, that I found things broken. These bounds no longer work:

```
impl<T: Decodable + Encodable> Msg<T> {
	fn new(bytes: Bytes) -> Result<Self> {
		let mut dec = Decoder::new(&buf);
        Ok(Self { msg: T::decode(&mut dec)? })
	}
}
```

Of course not, `Decodable` now has a lifetime. But quickly adding a lifetime surfaces the issue:

```
51 | impl<'a, T: Decodable<'a>> Msg<T> {
   |      -- lifetime `'a` defined here
52 |     fn new(buf: Bytes) -> anyhow::Result<Self> {
53 |         let mut dec = Decoder::new(&buf);
   |                       -------------^^^^-
   |                       |            |
   |                       |            borrowed value does not live long enough
   |                       assignment requires that `buf` is borrowed for `'a`
```
