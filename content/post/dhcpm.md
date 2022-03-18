---
title: "Rust superpowered DHCP cli with rhai scripts"
date: 2022-03-17T14:06:34-04:00
draft: false
---

I've been working on cli tool for a little while called `dhcpm` ("m" for "mock"). It started as a cli tool for constructing & sending arbitrary DHCP messages. I had been looking for a tool that could build various dhcp (mostly v4) messages with different parameters easily, then simply print the responses back so I could inspect its contents.

I discovered that you can use `nmap` scripts for this, and there are a 2 pre-written dhcp scripts in a typical nmap install. For example, to send a discover message:

```
sudo nmap -sU -p 67 --script=dhcp-discover <target>
```

(`U` for UDP, `67` is the default dhcpv4 port, the script we want to run and the IP of the dhcp server)

See [the docs](https://nmap.org/nsedoc/scripts/dhcp-discover.html) for arguments. While this works for basic tests, it didn't have all the features that I wanted out of the box... and I suck at writting lua... and with [dhcproto](https://github.com/bluecatengineering/dhcproto) all I really had to do was generate a cli parser from a struct and write some bytes to a UDP socket. So, while nmap is super flexible, I do still feel there is room for a tool focused just on DHCP.


## CLI

Hacking something workable came together pretty quickly, `dhcpm` currently looks like this:

```
> dhcpm --help

Usage: dhcpm <target> [-b <bind>] [-p <port>] [-t <timeout>] [--output <output>] [--script <script>] [--no-retry <no-retry>] [<command>] [<args>]

dhcpm is a cli tool for sending dhcpv4/v6 messages

ex  dhcpv4:
        dhcpm 0.0.0.0 -p 9901 discover  (unicast discover to 0.0.0.0:9901)
        dhcpm 255.255.255.255 discover (broadcast discover to default dhcp port)
        dhcpm 192.168.0.1 dora (unicast DORA to 192.168.0.1)
        dhcpm 192.168.0.1 dora -o 118,C0A80001 (unicast DORA, incl opt 118:192.168.0.1)
    dhcpv6:
        dhcpm ::0 -p 9901 solicit       (unicast solicit to [::0]:9901)
        dhcpm ff02::1:2 solicit         (multicast solicit to default port)

Positional Arguments:
  target            ip address to send to

Options:
  -b, --bind        address to bind to [default: INADDR_ANY:0]
  -p, --port        which port use. [default: 67 (v4) or 546 (v6)]
  -t, --timeout     query timeout in seconds [default: 5]
  --output          select the log output format (json|pretty|debug) [default: pretty]
  --script          pass in a path to a rhai script
                    (https://github.com/rhaiscript/rhai) NOTE: must compile
                    dhcpm with `script` feature
  --no-retry        setting to "true" will prevent re-sending if we don't get a
                    response [default: false]
  --help            display usage information

Commands:
  discover          Send a DISCOVER msg
  request           Send a REQUEST msg
  release           Send a RELEASE msg
  inform            Send an INFORM msg
  dora              Sends Discover then Request
  solicit           Send a SOLICIT msg (dhcpv6)
```

There are some base parameters that tell the tool which ports to bind to, the target, etc, there are even `output` options to tell `tracing` to log structured JSON or more readable logs. The meat of it are the subcommands for each of the dhcpv4 message types, with DHCPv6 is unfinished at the moment. Subcommands each have their own parameters:

```
> dhcpm 0.0.0.0 discover --help 

Send a DISCOVER msg

Options:
  -c, --chaddr      supply a mac address for DHCPv4 [default: first avail mac]
  --ciaddr          address of client [default: None]
  -r, --req-addr    request specific ip [default: None]
  -g, --giaddr      giaddr [default: 0.0.0.0]
  --subnet-select   subnet selection opt 118 [default: None]
  --relay-link      relay link select opt 82 subopt 5 [default: None]
  -o, --opt         add opts to the message [ex: these are equivalent-
                    "118,hex,C0A80001" or "118,ip,192.168.0.1"]
  --params          params to include: [default: 1,3,6,15 (Subnet, Router,
                    DnsServer, DomainName]
  --help            display usage information
```

With this arbitrary DHCP options can be set with hex or an ip string (ex. `--opt 118,ip,192.168.0.1`), we can change some IPs in the header like `giaddr`/`ciaddr`, the hardware address (`chaddr`), etc. This was great for quick testing with different parameters. But what about nmap's scripting engine? While you can use `dhcpm` inside a shell script, sometimes that can be a bit unwieldy when we're talking about poking into a text formatted DHCP message to pull out fields to pass to subsequent runs. It would be cool to have a mini scripting engine to edit scripts on the fly, change a couple parameters on what might be a chain of several messages and re-run.

## Scripting

At this point I stumbled on [rhai](https://github.com/rhaiscript/rhai), an embedded scripting environment for Rust. What attracted me to it was after looking at the code examples, it seemed like I already "knew" rhai. It looks much less foreign to me than lua. It's basically Rust with dynamic types, and it has a well documented [book](https://rhai.rs/book/) with some good examples of how to integrate with a Rust application. Let's take a look at the integration with `dhcpm`.

In `rhai` you can create a new [Engine](https://docs.rs/rhai/latest/rhai/struct.Engine.html) that can do things like call `engine.eval::<T>("1 + 2")`. What's passed will be executed by `rhai` and have its result returned. On other hand, you can pass a path to the engine and execute an entire script. This is what `dhcpm` does with the path provided through the cli.

Rhai also provides a robust set of macros for providing Rust code to the running script

```
#[export_module]
pub mod discover_mod {
    use tracing::trace;
    #[rhai_fn()]
    pub fn args_default() -> DiscoverArgs {
        DiscoverArgs::default()
    }
    #[rhai_fn(global, name = "to_string", name = "to_debug", pure)]
    pub fn to_string(args: &mut DiscoverArgs) -> String {
        format!("{:?}", args)
    }
    // ciaddr
    #[rhai_fn(global, get = "ciaddr", pure)]
    pub fn get_ciaddr(args: &mut DiscoverArgs) -> String {
        args.ciaddr.to_string()
    }
    #[rhai_fn(global, set = "ciaddr")]
    pub fn set_ciaddr(args: &mut DiscoverArgs, ciaddr: &str) {
        trace!(?ciaddr, "setting ciaddr");
        args.ciaddr = ciaddr.parse::<Ipv4Addr>().expect("failed to parse ciaddr");
    }
    ...
}
```

I think of this like an FFI, we're generating bindings for rhai to use. Some things to note, `rhai` can have any valid Rust type in functions exposed to it, although it looks to me like anything that is a custom type needs to have any variants/methods/etc explicitly exposed for it to be useful in rhai. Notice also that rhai's first paramater takes by `&mut Thing`. All methods can mutate; this is a scripting language after all. In any case, this can be `registered` with the `Engine`

```
engine
    .register_type_with_name::<DiscoverArgs>("DiscoverArgs")
    .register_static_module(
        "discover",
        exported_module!(crate::discover::discover_mod).into(),
    );
```

And using it in rhai:
```
let args = discover::args_default();
args.ciaddr = "1.2.3.4";
print(args)
```

This made it all fairly mechanical to expose the different configuration structs to rhai so that they could be created and modified inside the script. The next issue was, how will I get rhai to actually send the dhcp message that can be built from these arguments?

`dhcpm` is a simple tool, at startup it creates one `Arc<UdpSocket>` and two threads, one to `recv` and one `send`. These are complemented by a pair of channels so provide some scallfolding for the message runner. 

```
// messages put on `send_tx` will go out on the socket
let (send_tx, send_rx) = crossbeam_channel::bounded(1);
// messages coming from `recv_rx` were received from the socket
let (recv_tx, recv_rx) = crossbeam_channel::bounded(1);

runner::sender_thread(send_rx, soc.clone());
runner::recv_thread(recv_tx, soc);
```

`send_tx` will put any dhcp messages sent to it on the socket. `recv_rx` will be forwarded any any messages we read from the socket, however there can only be one reading at a time, otherwise we won't know which message is meant for which receiver. These channels passed into `TimeoutRunner` when it is initialized which I won't post the code for, but suffice to say it sends a messages and exits with an error if there is no reply within a certain time period or if we get a `SIGINT`.

For creating a rhai binding to this, initially I didn't understand how that would work. What's needed is a function that will take the `DiscoverArgs` and an already initialized value (of type `TimeoutRunner`), but one provided from the Rust environment, not the script. However, with function macros I showed before, all parameters were provided by Rhai, there is no way to pass in an already initialized variable.

I initially thought of using something like `Lazy` or `OnceCell` and putting `TimeoutRunner` there, but it didn't feel right. Eventually, I stumbled over closure bindings for `engine` and that appeared to solve this problem.

```
let run = runner.clone();
...
engine.register_fn("send", {
    move |args: &mut DiscoverArgs| {
        let mut new_runner = run.clone();
        // replace runner args so it knows which message type to run
        new_runner.args.msg = Some(MsgType::Discover(args.clone()));
        new_runner.send().expect("runner failed").unwrap_v4()
    }
})
```

This will attach the method `send` on `DiscoverArgs` allowing the script to use it like this:

```
let args = discover::args_default();
args.ciaddr = "1.2.3.4";
print(args);
let msg = args.send();
```

Currently in `dhcpm` there are bindings for discover, request, inform & release, and getters/setters for each, allowing one to script the full DORA negotiation for an IP with custom fields for each.

## Conclusion

While what I've described in this post might not be banal to some folks, there was something about it that I found exciting and made me want to share. I think there are so many possibilities for this type of a setup where you can embed a scripting language with superpowers provided by Rust into your application.

