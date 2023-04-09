---
title: "Setting up a WireGuard VPN"
date: 2021-09-28T17:53:09Z
---

In this post I want to give a quick rundown of the few steps required to use
[WireGuard](https://www.wireguard.com/) as a VPN. My setup uses a Raspberry Pi
running [Arch Linux ARM](https://archlinuxarm.org/) as the main gateway into my
home network. I'll configure another peer such that it can connect to the Pi
and thus other devices in my network. The setup is IPv4-only at the moment
because my ISP sucks. Also you should have some prior knowledge in networking.

## First steps

As ArchLinux ARM (in its default configuration) ships with a Linux kernel with
WireGuard support enabled, the first step is to install WireGuard's userland
tools.

    $ pacman -S wireguard-tools

Naturally the package is not called `wireguard-tools` on every platform. A
complete list of packages for different operating systems can be found
[here](https://www.wireguard.com/install/). This gives you access to the `wg`
utility, which can perform several management tasks and the `wg-quick` utility,
which can load and apply configurations from files. I'll not be making much use
of in-place configuration and instead jump directly into writing configuration
files, as they are pretty straightforward regardless. All configuration files
live in `/etc/wireguard`. They could be located anywhere but this path allows
shorthand notation in `wg-quick` arguments.

### Server

First let's set up the server (i.e. the Raspberry Pi), which the client can
then connect with in order to have a tunneled connection into my home network.
Create a configuration file in `/etc/wireguard/` called `wg0.conf`. Set its
mode to `0600` because it will contain a private key and therefore shouldn't be
world-readable. The configuration syntax is somewhat similar to Windows' INI
files. The server's interface is configured like this:

    [Interface]
    Address = 192.168.42.1/24
    ListenPort = 50040
    PrivateKey = RG9udCB1c2UgdGhpcyB2YWx1ZSB5b3UgZHVtYmFzcyE=
    MTU = 1420

`Address` refers to the server's address within the WireGuard tunnel. In my
setup I wanted to have the WireGuard "network" live under the netmask
`192.168.42.0/24`. Having the main gateway be `192.168.42.1` makes things
simple to understand. `ListenPort` is `50040` but can be anything of course
(I'm not even sure there is a definite default yet). Setting `MTU` to `1420` is
the default and should work pretty much everywhere. Most interesting is the
`PrivateKey` field. WireGuard uses Ed25519 keys for authentication and this is
simply the server's identity. The value can be generated via `wg genkey`.

And that's is on the server side for now. You can call `wg-quick up wg0` to
enable this interface right now and verify its existence via the output of `ip
link` and `ip address` commands.

### Client

Now for the same on the client.

    [Interface]
    Address = 192.168.42.2/24
    ListenPort = 50041
    PrivateKey = TmV2ZXIgZXZlciBjb3B5IGtleXMgZnJvbSBndWlkZXM=
    MTU = 1420

No surprises here. The client also has a private key and its IP is to be
`192.168.42.2`. The `ListenPort` should be different to the server's port, as
WireGuard should be able to establish connections in both directions.

### Peering

Now we'll connect client and server. To make this work we'll need to exchange
keys, as the server needs to know the client's public key and vice versa. The
command `wg pubkey` can be used to derive the public key from the private key.
For example, to get the server's public key:

    $ echo "RG9udCB1c2UgdGhpcyB2YWx1ZSB5b3UgZHVtYmFzcyE=" | wg pubkey
    QXJlIHlvdSByZWFkaW5nIHRoaXM/IEZvciByZWFsPyA=

(Sidenote: This will write the private key into your shell history. So you may
want to write the key into a file instead and `cat` it's contents into `wg
pubkey`)

While not strictly required, you may also generate and exchange a _pre-shared
key_ between the peers, such that you also benefit from a layer of symmetric
cryptography in case you want to harden against quantum cryptanalysis. Such
a key can be generated via `wg genpsk`:

    $ wg genpsk
    eW91IGNvdWxkIGFjdHVhbGx5IHVzZSB0aGlzIG9uZSA=

Both the client's and the server's configuration needs an additional `[Peer]`
section now.

For the server this section needs to look like this:

    [Peer]
    PublicKey = a2V5c21hc2hrZXlzbWFzaGtleXNtYXNoa2V5c21hc2g=
    PresharedKey = eW91IGNvdWxkIGFjdHVhbGx5IHVzZSB0aGlzIG9uZSA=
    AllowedIPs = 192.168.42.2/32

And for the client like this:

    [Peer]
    PublicKey = QXJlIHlvdSByZWFkaW5nIHRoaXM/IEZvciByZWFsPyA=
    PresharedKey = eW91IGNvdWxkIGFjdHVhbGx5IHVzZSB0aGlzIG9uZSA=
    AllowedIPs = 192.168.42.1/32
    Endpoint = vpn-host.example:50040

Notice the additional `Endpoint` value in the client. This is because the
client obviously needs to know where the server is located such that a
WireGuard tunnel can be established. This does not need to be a domain name and
could instead just be a raw IP address. Of course, in a VPN setup there is no
way we could know an `Endpoint` value for the client. The server will learn the
client's endpoint after each handshake, which is implicitly performed whenever
the client starts to send data to the server.

...aaand that's it! Do `wg-quick up wg0` on both devices and try to perform a
ping over the WireGuard tunnel. You can inspect the state of the tunnel via:

    $ wg

## VPN

Our devices can now talk to _each other_ over WireGuard. But that is not
enough, as the aim is to allow routing traffic into my home network. I don't
care about routing connections to the internet over WireGuard and simply want
my client to be able to access devices on the `192.168.0.0/24` network (i.e. my
home network).

We're way more than halfway there. The last two puzzle pieces are: IP
forwarding, routing and having traffic from the client to `192.168.0.0/24` move
through WireGuard.

### IP Forwarding

On Linux, routing can be enabled through `sysctl`:

    $ sysctl -w net.ipv4.ip_forward=1

To make this setting stick at boot, write this setting into a file in the
directory `/etc/sysctl.d`:

    $ echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/ip-forwarding.conf

### Routing

Routing, or to be more precise _masquerading_, can be enabled via `iptables`:

    $ iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

`eth0` needs to be replaced with the canonical name of your server's network
interface.

This can also be automated via WireGuard's configuration manager, which is able
to execute commands when an interface is enabled and disabled. Add the command
into the `PostUp` option in the `[Interface]` section:

    [Interface]
    (...)
    PostUp = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

`PostDown` removes this when the WireGuard interface is disabled.

### Traffic to 192.168.0.0/24

This is added to the client's configuration. Remember the `AllowedIPs` key in
the `[Peer]` section? You can simply add the whole network like this:

    [Peer]
    (...)
    AllowedIPs = 192.168.42.1/32, 192.168.0.0/24

That's it. `wg-quick` will set up the routes accordingly.

## And?

That's it. You're done. Enjoy your VPN :)

## Persistent Keepalive

This is a small update after a few months of very successfully using WireGuard.
You might find yourself in the following situation: Consider that you have two
devices, _A_ and _B_, on your network. _A_ has the address `192.168.42.2` and
_B_ has the address `192.168.42.3`. Your router and gateway is at
`192.168.42.1`.  `wg-quick` sets routing up for you, simply sending all traffic
towards `192.168.42.0/24` over your router. Sure you _could_ configure a direct
connection between each and every peer manually, but this would get super
annoying super fast.

Device A might be... whatever. And device B might be some gizmo that you only
boot up sporadically via Wake-on-LAN. You'll find that, once B is booted up, A
has no idea how to talk to B. The router doesn't know that B is awake yet. And
B never had any reason to communicate with the router. So the router won't have
any clue how to route A's traffic to B. Remember how WireGuard is advertised as
not being a talky protocol by default? This is exactly that principle in action
and in most cases its perfectly fine. However here it falls flat on its face.
What we need to do here, is make sure that the router always knows how to talk
to B and that it maintains a route.

For this end, we can simply add the following line to _B_'s `wg.conf`:

    [Peer]
    (...)
    PersistentKeepalive = 30

Now B will say "hello" to the router every 30 seconds, thus allowing the router
to know of B's existence. You can, of course, also choose a higher interval.
Most important is the initial handshake from B to the router right after B has
finished booting up.

And this concludes one of the few cases in which you should add
`PersistentKeepalive` to your WireGuard configurations. Seriously, if you
don't encounter any issues just leave it out.

