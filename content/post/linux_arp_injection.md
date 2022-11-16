---
title: "ARP Injection in Rust on Linux"
date: 2022-11-16T00:13:15-04:00
draft: false
---

Don't ask why, but I had to learn about this a while ago and had been meaning to write it down.

ARP is [Address Resolution Protocol](https://en.wikipedia.org/wiki/Address_Resolution_Protocol), it is a layer 2 protocol used in local IPv4 networks for discovering a MAC address for a given IP. Say you want to send a packet to an IP on your local network, `192.168.0.136`. The ethernet frame requires a destination MAC address, but you don't know what it is yet. This is discovered either from the local routing table or by ARP. If your routing table doesn't contain `192.168.0.136`, an ARP broadcast (where the destination MAC is set to FF:FF:FF:FF:FF:FF) is sent. `192.168.0.136` would then respond via ARP unicast with its MAC & IP. This entry can be added to the cache for future use.

Typically, clients will do an ARP broadcast when their IP or MAC changes so other hosts on the network can update their internal mapping of IP to MAC. This all changed in IPv6, v6 networks don't use ARP in favor of Neighbor Discovery Protocol (NDP), part of ICMPv6.

Fun fact, you can check out your ARP cache on Linux by running `cat /proc/net/arp`:

```
IP address       HW type     Flags       HW address            Mask     Device
192.168.0.136    0x1         0x2         xx:xx:xx:xx:xx:xx     *        enp6s0
192.168.0.1      0x1         0x2         xx:xx:xx:xx:xx:xx     *        enp6s0
```

## Stumbling through C

This all began after I stumbled on a bit of code when looking through the source for `dnsmasq`. A DHCP server has a unique case being the thing that assigns IPs to devices on a network. Without going into too much detail, sometimes a server will want to unicast a response back to a particular client before the IP has been fully assigned. Meaning the server has both the destination IP and MAC, but the routing table doesn't have that information yet. In this case the server can inject the entry into the table before sending the packet via unicast.

As a sidenote, I think there is another way for the DHCP server to do this where it constructs the raw ethernet frame with the MAC included instead of injecting into the cache, but don't take that to the bank. Here is the snippet from `dnsmasq`:

```C
	/* unicast to unconfigured client. Inject mac address direct into ARP cache.
	struct sockaddr limits size to 14 bytes. */
	dest.sin_addr = mess->yiaddr;
	dest.sin_port = htons(daemon->dhcp_client_port);

	memcpy(&arp_req.arp_pa, &dest, sizeof(struct sockaddr_in));

	arp_req.arp_ha.sa_family = mess->htype;

	memcpy(arp_req.arp_ha.sa_data, mess->chaddr, mess->hlen);

	/* interface name already copied in */
	arp_req.arp_flags = ATF_COM;
	if (ioctl(daemon->dhcpfd, SIOCSARP, &arp_req) == -1)
	my_syslog(MS_DHCP | LOG_ERR, _("ARP-cache injection failed: %s"), strerror(errno));
```

[here](https://github.com/imp/dnsmasq/blob/770bce967cfc9967273d0acfb3ea018fb7b17522/src/dhcp.c#L419)

Pretty dense, but we can learn some interesting things from this. The code puts the destination IP and port in the `arp_pa` field of `arp_req`, it sets the hardware type (`mess->htype` will likely be the code for "Ethernet" in this case), it sets the MAC (`mess->chaddr`) and a flag `ATF_COM` (stands for "Lookup complete", apparently). Finally, it uses the syscall `ioctl` passing the `SIOCSARP` parameter with a file descriptor for a socket and a pointer to the struct.

`ioctl` stands for "i/o control". It's kind of a grab-bag syscall that does a lot of stuff depending on its params. I found the shape of `arp_req` by looking up the `arp` linux module [here](https://manpages.courier-mta.org/htmlman7/arp.7.html)

```C
struct arpreq {
	struct sockaddr arp_pa;
	/* protocol address */
	struct sockaddr arp_ha;
	/* hardware address */
	int arp_flags;
	/* flags */
	struct sockaddr arp_netmask;
	/* netmask of protocol address */
	char arp_dev[16];
};
```

And right below that:

> SIOCSARP, SIOCDARP and SIOCGARP respectively set, delete, and get an ARP mapping. Setting and deleting ARP maps are privileged operations and may be performed only by a process with the CAP_NET_ADMIN capability or an effective UID of 0.

There's a command to modify the arp cache on linux aptly named `arp`, which I believe was supplanted later by `ip neigh` because the latter works with ipv6 also. In any case, the source for the original `arp` uses `ioctl` similarly to `dnsmasq` to add entries. [Source here](https://github.com/ecki/net-tools/blob/master/arp.c#L266) and you can try `arp -s 192.168.0.1 -i ethX xx:x:xx:xx:xx:xx` to add an entry, `arp -d 192.168.0.1` deletes. Under the hood, these both run `ioctl(fd, SIOCSARP, &arp_req)` & `ioctl(fd, SIOCDARP, &arp_req)`, respectively.

So all that code is really just calling "set" with a specific IP:port for the ARP cache.

## In Rust

```rust
pub struct arpreq {
	pub arp_pa: ::sockaddr,
	pub arp_ha: ::sockaddr,
	pub arp_flags: ::c_int,
	pub arp_netmask: ::sockaddr,
	pub arp_dev: [::c_char; 16],
}
```

Translating the `dnsmasq` snippet as best I could to Rust left me with:

```rust
let addr_in: libc::sockaddr_in = libc::sockaddr_in {
	sin_family: libc::AF_INET as _,
	sin_port: port.to_be(),
	sin_addr: unsafe { *(&yiaddr as *const _ as *const _) },
	..unsafe { std::mem::zeroed() }
};
// memcpy to sockaddr for arp_req
let arp_pa: libc::sockaddr = unsafe { std::mem::transmute(addr_in) };
// create arp_ha (for hardware addr)
let arp_ha: libc::sockaddr = libc::sockaddr {
	sa_family: htype as _,
	sa_data: unsafe {
		let mut sa_data = [0; 14];
		let len = chaddr.len();
		sa_data[..len].copy_from_slice(std::mem::transmute::<_, &[libc::c_char]>(chaddr));
		sa_data
	},
};
let arp_req = libc::arpreq {
	arp_pa,
	arp_ha,
	arp_flags: libc::ATF_COM,
	..unsafe { std::mem::zeroed() }
};

let res = unsafe {
	libc::ioctl(
		soc.as_raw_fd(),
		libc::SIOCSARP,
		&arp_req as *const libc::arpreq,
	)
};
if res == -1 {
	return Err(io::Error::last_os_error());
}
```

If you run this code with some values for `yiaddr`/`port`/`htype` & `chaddr` then `cat /proc/net/arp` you should see an entry populated in the table.

Lots of unsafe here, but that's to be expected given we are using the C FFI. I found that I had to first put the IP to-be-injected (`addr_in`) as a `sockaddr_in` then transmute to `sockaddr` in order to set `arp_pa` properly. Of note, `sa_data` is an array of `char`. I didn't realize at first that `char` is signed or unsigned depending on the architecture. I figured this out the hard way by using `i8` at first and having this code explode on ARM where `char` is unsigned. Switching the Rust code to use `c_char` fixed that. This was pointed out by a helpful user on discord whose handle I have forgotten (thank you!).

Another option for calling `ioctl` would be to use the `nix` crate. Because the surface area of `ioctl` is so broad, `nix` has macros that generate functions for calling `ioctl`. After some futzing, I found that this worked:

```
nix::ioctl_write_ptr_bad!(siocsarp, nix::libc::SIOCSARP, nix::libc::arpreq);
```

This generates a function called `siocsarp` that takes a `arpreq`. You can call it like:

```rust
if let Err(err) = unsafe { siocsarp(fd, &arp_req as *const _) } { // call the macro generated function
	// error
}
```

I'm not particularly sure if `ioctl`'s ARP syscalls need a certain socket type, but a UDP socket seems to work. As I was figuring all this out, I was tipped off several times that this method of setting the ARP cache is outdated. Firstly, the `arp` manpage says:

> There is also a mechanism for managing the ARP cache in user-space by using netlink(7) sockets

Secondly, the `ip neigh add` method of adding entries uses netlink sockets. Only the anachronistic `arp` uses `ioctl`.

I'll save learning about netlink sockets for another day, however. If it's good enough for dnsmasq... :)
