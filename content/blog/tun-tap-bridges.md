+++
title = "TUN, TAP, Bridges"
date = 2024-09-17
+++

I was recently exploring [firecracker](https://firecracker-microvm.github.io/), a very minimal VM manager, for use in a
home lab setup to make provisioning new VMs very quick. When reading the documentation, I saw that the guest networking
is exposed as a TAP device to the host operating system. I know a little bit about TUN/TAP devices, but most of my
experience has been running various `ip` commands and tinkering with things until they seem to be working. I wanted to
dive in a little more to see how various virtual networking things worked under the hood as I really had no idea!

# Network Interfaces
First off, what actually is a network interface in linux? Basically a network interface is a
software representation of a physical or virtual network device. If you've ever run a command like `ip a` or `ifconfig`,
you've seen your network interfaces listed out.  Usually when you are writing a basic networked application you don't
actually care about the interfaces directly - you would just create a socket and bind/connect/do whatever you want.
Where the traffic ingresses/egresses the machine is dependent on the kernel's routing policies and  routing table (check
out `ip rule list` and `ip route list`). This all seems straightforward enough for a physical interface, when I list it
out with `ip a` I can see my NIC's MAC address and other information, which maps cleanly in my mind to a physical
component of my computer. So what are the virtual interfaces exactly?

# Virtual Interfaces
It turns out that sometimes we want something that looks and acts like a physical network interface
but we control it from software without having to hack around in the kernel's network stack directly. This is where
virtual interfaces are useful. Virtual interfaces are commonly used for VPN software and virtualization/container
networking. Probably the most common virtual interface that people are familiar with is the loopback interface. When
we send something to `127.0.0.1`, the packet eventually reaches the loopback interface. We can see the code for it
[here](https://github.com/torvalds/linux/blob/a430d95c5efa2b545d26a094eb5f624e36732af0/drivers/net/loopback.c#L69-L93),
basically rather than the packet being written to a physical network device, it's just "looped" back into the kernel's
networking stack with the call to [`__netif_rx`](https://docs.kernel.org/networking/kapi.html#c.netif_rx).

Cool, what about other types of virtual interfaces? Specifically, I have been looking at
[TUN/TAP](https://docs.kernel.org/networking/tuntap.html) interfaces. Here's the definition from the docs:

> TUN/TAP provides packet reception and transmission for user space programs. It can be seen as a simple Point-to-Point
or Ethernet device, which, instead of receiving packets from physical media, receives them from user space program and
instead of sending packets via physical media writes them to the user space program.

Seems straightforward enough, a TUN/TAP device is a virtual networking interface that a userspace process simply reads
from and writes to. Note that TUN/TAP are two different types of devices, a TUN devices works at the IP layer, whereas a
TAP devices works at the ethernet layer. When a userspace process attaches to a TUN/TAP device it simply receives the
raw packets/frames routed to that interface and does whatever it wants with them. When a userspace process writes a
packet/frame to a TUN/TAP devices, that packet/frame is simply injected into the kernel's network stack where it will be
directed by the routing policies and routing tables, similar to if the packet/frame was received from a physical
interface.

# TUN Use Case

So how does this all work in something like a VPN daemon, using a TUN device? Let's look at how we'd implement a simple
peer to peer connection in our hypothetical VPN daemon. The daemon would start up, create the TUN interface (if it
didn't already exist) attach itself to it, then bring the TUN interface up. Note that a TUN/TAP interface will only be
up if a process is attached to it, otherwise the packets/frames have nowhere to go! We would then configure our routing
so traffic we want to go through the VPN is routed through our TUN interface. If we were using a VPN for a more
privacy-oriented use case (e.g. we want all our traffic to go through the VPN tunnel, where a VPN server we are
connected to that hopefully isn't keeping logs will send our traffic out to the internet, then send the response back
through the tunnel to us) then we would route all of our internet traffic through the TUN interface. If we were
using a VPN to connect to, let's say a remote network under the 10.0.0.0/24 subnet, we would update our routing tables
to direct traffic heading to 10.0.0.0/24 to our TUN interface. The daemon would open a normal socket (usually UDP for
VPNs) with the other side being our remote VPN machine, running the same daemon. Traffic local to our machine that is
routed through the TUN interface will be received by the daemon running on the same machine. The daemon will take that raw,
unencrypted traffic, encrypt it and do whatever other VPN stuff, then send it over the socket it created to the remote
VPN machine. The remote VPN machine receives that traffic, where it goes through the normal networking paths and ends up
in the socket where the VPN daemon is listening. It then decrypts the traffic and writes it to the TUN interface, which
effectively writes it into the kernel's networking stack as if we received a normal, non-VPN encrypted, packet.

If I didn't explain this well, here is an example from the [TUN/TAP docs
FAQ](https://docs.kernel.org/networking/tuntap.html):

> Let’s say that you configured IPv6 on the tap0, then whenever the kernel sends an IPv6 packet to tap0, it is passed to
the application (VTun for example). The application encrypts, compresses and sends it to the other side over TCP or UDP.
The application on the other side decompresses and decrypts the data received and writes the packet to the TAP device,
the kernel handles the packet like it came from real physical device.

And another from a [good blog](https://lxd.me/a-simple-vpn-tunnel-with-tun-device-demo-and-some-basic-concepts) I found
explaining this use case:

> In a nutshell, the process for client side tunneling is:
> 1. Open an UDP socket whose other side is the server.
> 2. Create the `tun` device, configure it and bring it up.
> 3. Configure routing table.
> 4. Read packets from `tun` device, encrypt, send to server via socket created in 1st step; And read from the socket,
decrypt, write back to `tun` device. This step goes on and on.

# TAP Use Case

Let's check out another use case, using a TAP device. This time rather than looking at a hypothetical VPN daemon, let's
look at Firecracker. We can see their brief design doc
[here](https://github.com/firecracker-microvm/firecracker/blob/main/docs/design.md), where it says:

> Firecracker emulated network devices are backed by TAP devices on the host. To make use of Firecracker, we expect our
> customers to leverage on-host networking solutions.

and

> Firecracker provides VirtIO/block and VirtIO/net emulated devices, along with the application of rate limiters to each
> volume and network interface to make sure host hardware resources are used fairly by multiple microVMs.

Ok, so Firecracker exposes a [virtio](https://wiki.osdev.org/Virtio) device to the guest OS, which is actually backed by
a TAP device in the host OS. Firecracker guests' packets are routed to the virtio device, which firecracker then writes
to the corresponding TAP device in the host OS, which are written into the host's networking stack as if it had received
them from a physical interface. From here, the host OS can be configured as necessary (via iptables or bridges, which we
will cover next) to handle how the packets are dealt with. This is where the "we expect our customers to leverage
on-host networking solutions" part from the above docs comes in. The virtio net device for firecracker is defined
[here](https://github.com/firecracker-microvm/firecracker/blob/a4b3d932ced21d1ae73cf5c1690cb746095bea2f/src/vmm/src/devices/virtio/net/device.rs#L105-L110).
And the actual implementation of writing frames from the guest to the host's TAP device in the method
`write_to_descriptor_chain`. It looks like the actual memory management is done by the Rust crate `vm_memory`, there is
a brief description of how the memory management is handled from their
[docs](https://github.com/rust-vmm/vm-memory/blob/54c67221bdfbc88a38e7671d4ef790f67bde3dee/DESIGN.md?plain=1#L89-L123),
it's an interesting little thing to read! Firecracker has some short guides on how you'd want to set up your hosts
networking in their [docs](https://github.com/firecracker-microvm/firecracker/blob/main/docs/network-setup.md) as well.

So why is a TAP device used here over a TUN? Because we are emulating hardware (that's what firecracker's virtio device
is doing), firecracker needs to work at the same level. The hardware works at the ethernet layer and so must our
virtualized device. TUN devices work great for situations like our VPN daemon above where we are just routing traffic,
but this is insufficient for emulating a hardware network interface! This is why we see TAP devices used more in
virtualization.

# Bridges
So far we've talked about TUN/TAP virtual devices and the loopback interface. Another commonly used virtual
network interface is a bridge device. I'm bringing this up because it's mentioned in the Firecracker [network setup
docs](https://github.com/firecracker-microvm/firecracker/blob/main/docs/network-setup.md) but isn't explained much
there, and I didn't know much about them.

A bridge interface just acts like a physical switch. Switches act at the ethernet layer - you send an ethernet frame to
it and if the device with the destination MAC address of the ethernet frame is connected to the switch, the frame gets
sent to that device. The switch itself keeps track of what MAC addresses are connected to which physical port so it
knows where to forward the traffic. A bridge interface is basically exactly the same, but virtual. Instead of physically
plugging a device in to it, you add an interface (like a TAP device or a physical one) and you can think of this like
plugging an ethernet cable from each interface into a physical switch. I haven't played with this yet in firecracker as
it seems some people have to [do some
tweaking](https://devopschops.com/blog/communicating-between-firecracker-microvms-using-bridges/) to get everything to
work correctly for VM-to-VM communication. I'll cross that bridge (ha) when I get there!

# Summary
- A network interface is a software representation of either a physical or virtual network device.
- Virtual interfaces act like physical ones but have some special logic in them to make them useful without having to
directly mess around in the kernel's network stack.
- TUN/TAP devices allow us to attach a program to in order to read/write raw ethernet frames/IP packets and process
them as we see fit.
- TUN devices work at the IP layer, TAP devices work at the ethernet layer.
- TUNs are commonly used for routing use cases, like VPN clients.
- TAPs are commonly used in virtualization as they work on the same layer as a hardware interface would (ethernet layer).
- Bridges are basically just virtual switches, we can add interfaces to the bridge and we can think of it like
connecting an ethernet cable from our interfaces to a physical switch.
