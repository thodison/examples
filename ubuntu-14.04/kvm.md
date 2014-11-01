> Back to [Table of Contents](https://github.com/jpfluger/examples)

# Ubuntu networking

These examples describe in detail different situations on how to configure networking on Ubuntu Server 14.04 with kvm-libvirt, Openvswitch, bridge-utils, bonding and vlans. I will try to shed light on what can be maddingly [confusing](https://bugs.launchpad.net/serverguide/+bug/1103870) subject matter, especially given that web-examples tend to only cover one-aspect of an implementation. This creates a situation when one reads through a 2nd tutorial, then a 3rd, 4th and 5th and they are all different but yet all correct configurations!  

The goal is to take the mystery out of Ubuntu networking by examples. This will help you understand Ubuntu's current networking options so that you can better marry that to your own project's networking requirements.

We will cover:

* Can the hardware run kvm?
* Identify hardware network interfaces
* Install additional packages
* Unraveling mysteries
* Summary of interface choice configurations
* eth0 (dhcp and static)
* eth0:0 (multiple IPs using aliases on a single network interface)
* eth1 (a second network interface)

## My test system

You do not need to configure a test system in order to follow along the examples below. But I describe my test environment so that results might be replicated, if so desired.

If you want to follow the examples, you may run the commands on (1) a clean install of Ubuntu Server 14.04 or (2) within a VM of Ubuntu Server. I chose the second option, although the first would work, so long as the test Server is pingable within its own network to verify network connectivity.

My laptop is the hypervisor using kvm-libvirt and a bridge. In a new VM, I installed Ubuntu Server 14.04. I also added a second network interface card to the VM.  During installation the setup wizard asks which networking interface should be primary. I chose `eth0` and left `eth1` unconfigured.  The install automatically assigned eth0 to use DHCP. This VM is the **Server** that all my tests will run against. In my examples below, I will configure this VM for networking and will turn it into a hypervisor. From within this Server, we will create a child VM. My laptop acts as the public internet. All three working together allow me to validate network connections and firewall rules.

Let me define the roles of the devices in my test system. I include synonyms for how the devices may be referenced in my examples.

* Laptop: root system hypervisor, acts as external client for ping tests, it's my test system version of the public internet
* VM Server: the "Server", the "Test Server", test VM hypervisor, most networking configurations done here, will create a child VM to use for testing with it. 
* Child VM: the "Host", the "Child Host", used to validate network connectivity and firewall rules

All commands issued within these examples either run on the VM Server or its child host instances. When I say ping the Server externally, I will be pinging the VM Server from the laptop because my laptop is in effect outside the visibility of the VM Server's sub-networks.

> Note: Typically I would have one single hypervisor at the Laptop level and not two. It is only to assist in my testing that I created the VM Server acting as a hypervisor.

## Post-install tasks

Ok, with a clean install of Ubuntu Server 14.04 just finished, ensure the system is up-to-date and an ssh server has been installed.

```bash
$ sudo apt-get dist-upgrade
$ sudo apt-get install openssh-server
```

Get the IP address of the running system.

```bash
$ ifconfig eth0 | grep inet
inet addr:10.10.11.248  Bcast:10.10.11.255  Mask:255.255.255.0
```

The server's currently configured IP is 10.10.11.248, which was allocated by DHCP.

From an external device, ping this Server.

```bash
$ $ ping -c 3 10.10.11.248
PING 10.10.11.248 (10.10.11.248) 56(84) bytes of data.
64 bytes from 10.10.11.248: icmp_seq=1 ttl=64 time=0.222 ms
64 bytes from 10.10.11.248: icmp_seq=2 ttl=64 time=0.326 ms
64 bytes from 10.10.11.248: icmp_seq=3 ttl=64 time=0.517 ms

--- 10.10.11.248 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2000ms
rtt min/avg/max/mdev = 0.222/0.355/0.517/0.122 ms
```

Good, now we are ready to get started. You may also now use `ssh` to connect to the Server.

```bash
$ ssh USER@10.10.11.248
```

But beware that during the course of these examples the ssh IP address will change and we will loose connectivity. When the IP address changes, external clients may complain of possible compromises to the ssh certificate. On the external client, run the following to remove the fingerprint from `known_hosts` and allow reassignment of the certificate using the new IP address.

```bash
$ ssh-keygen -R 10.10.11.248
$ ssh USER@NEW-IP-ADDRESS
```

## Can the hardware run kvm?

Answer this question: "Can this hardware or VM instance run kvm?" The answer can be found in Ubuntu's [pre-installation checklist](https://help.ubuntu.com/community/KVM/Installation).

Run this command.

```bash
$ sudo egrep -c '(vmx|svm)' /proc/cpuinfo
4
```

If the command returns 1 or more, then the CPU supports hardware virtualization and make sure it's enabled in the BIOS.

Install cpu-checker.

```bash
$ sudo apt-get install cpu-checker
```

Run kvm-ok.

```bash
$ kvm-ok
INFO: /dev/kvm exists
KVM acceleration can be used
```

## Identify hardware network interfaces

For most devices, network connectivity begins by enabling WIFI or plugging in a physical connecter (eg RJ-45 jack) to a network card.

Ubuntu uses [udev](http://manpages.ubuntu.com/manpages/trusty/man7/udev.7.html) to know manage software device events, such as mapping physical hardware to software.

Get a list of physical hardware attached to this Test Server.

```bash
$ lspci

# OR filter by term (e.g. "ethernet" or "wireless")... case insensitive search
$ lspci | grep -i ethernet
$ lspci | grep -i wireless
```

In the list you will see a hardware address followed by a description. Here are my lspci values for the two network card interfaces associated with the Test Server:

```
00:03.0 Ethernet controller: Red Hat, Inc Virtio network device
00:04.0 Ethernet controller: Red Hat, Inc Virtio network device
```

Now in the udev logs, we can search for devices mapped to interfaces by the pci number associated with the device.

```bash
$ grep -i \(net\) /var/log/udev | sort -u
KERNEL[0.800126] add      /devices/pci0000:00/0000:00:03.0/virtio0/net/eth0 (net)
KERNEL[0.800138] add      /devices/pci0000:00/0000:00:04.0/virtio1/net/eth1 (net)
KERNEL[0.800863] add      /devices/virtual/net/lo (net)
UDEV  [0.851846] add      /devices/pci0000:00/0000:00:03.0/virtio0/net/eth0 (net)
UDEV  [0.869047] add      /devices/pci0000:00/0000:00:04.0/virtio1/net/eth1 (net)
UDEV  [0.905996] add      /devices/virtual/net/lo (net)
```

These results show we have interfaces for `lo`, `eth0` and `eth1`. Notice the pci device points grep'ed from lspci are within the device path. 

> Note: If anyone knows a better one-liner command that gives me physical-to-interface results, please let me know!

To explore network hardware details, please see the examples on [linuxnix.com](http://www.linuxnix.com/2013/06/find-network-cardwiredwireless-details-in-linuxunix.html). Some of these tools only report back desired results on the root device and not within a Virtual Machine instance.

## Install additional packages

Before we install any packages, let's run a sanity check and see what interfaces are **expected** to run on our current Test Server. 

```bash
$ cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet dhcp
```

We are expecting `lo` and `eth0` to be running. Are they?

```bash
$ sudo ifconfig
eth0      Link encap:Ethernet  HWaddr 52:54:00:87:a5:22  
          inet addr:10.10.11.248  Bcast:10.10.11.255  Mask:255.255.255.0
          inet6 addr: fe80::5054:ff:fe87:a522/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:1011 errors:0 dropped:0 overruns:0 frame:0
          TX packets:717 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:116505 (116.5 KB)  TX bytes:119967 (119.9 KB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

Yes, they are present and the link is in an "UP" state.

---

Install these Openvswitch packages.

```bash
$ sudo apt-get install openvswitch-switch openvswitch-common
```

Install the following packages for kvm.

```bash
$ sudo apt-get install qemu-kvm libvirt-bin bridge-utils ubuntu-vm-builder qemu-system
```

> Note: I included qemu-system, which provides emulation binaries for other architectures. If you develop for embedded systems, you will need this installed.

Add to user groups.

```bash
$ sudo adduser `id -un` libvirtd
$ sudo adduser `id -un` kvm
```

Reboot.

```bash
$ sudo reboot now
```

Log back in.

## Unraveling mysteries

Remember the sanity check we ran before?  Let's see which networking interfaces are available **now**.

```bash
$ ifconfig

# My Results: Truncated
eth0      Link encap:Ethernet  HWaddr 52:54:00:87:a5:22  
          inet addr:10.10.11.248  Bcast:10.10.11.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0

virbr0    Link encap:Ethernet  HWaddr b2:ed:68:06:7c:ec  
          inet addr:192.168.122.1  Bcast:192.168.122.255  Mask:255.255.255.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
```

As seen, the `virbr0` interface has been added.  Where has this interface been defined? Is it in the `interfaces` file?

```bash
$ cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet dhcp
```

No, `virbr0` has not been defined there. What is going on?

We need to use a couple of other commands to view the system modifications that happened.

Have any bridges been configured?

```bash
$ brctl show
bridge name	bridge id		STP enabled	interfaces
virbr0		8000.000000000000	yes	
```

Yes, one bridge entry. Okay, so we know that this `virbr0` interface is tied to a bridge named `virbr0`. 

Has the routing table been modified?

```bash
$ route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         10.10.11.1      0.0.0.0         UG    0      0        0 eth0
10.10.11.0      *               255.255.255.0   U     0      0        0 eth0
192.168.122.0   *               255.255.255.0   U     0      0        0 virbr0
```

Yes, a virbr0 entry has been added to the routing table, as well.

I'll repeat: what's going on here?

Let me take a step back in time. Before libvirt was installed, `virbr0` did not exist. We only had interfaces for loopback and eth0.  `virbr0` means "virtual bridge 0" and was automatically created by libvirt during installation. `virbr0` was configured as a NAT-only interface. This means virtual machine hosts that use this bridge can get out to the network via the `eth0` interface but any devices on the other side cannot initiate requests into virbr0 clients.

I would suggest reading through this libvirt wiki on [Virtual Networking](http://wiki.libvirt.org/page/VirtualNetworking).

If you are a newbie, that [Virtual Networking](http://wiki.libvirt.org/page/VirtualNetworking) read was probably pretty heavy stuff. If you aren't a newbie, you might be even more confused than you are now. Why does libvirt create a `virtual bridge` when it could be using the standard linux `brctl` to create and edit bridges?  The libvirt virtual bridge allows libvirt to offer more features and interactivity than the standard Linux networking and `brctl` command. It adds NAT, DNSMASQ and iptables rules for us, alleviating work for creating certain configurations. It even creates a DHCP pool for connected VM hosts within the virbr0 subnet.

Because libvirt adds extra networking features, it creates virtual bridge files in `/etc/libvirt/qemu/networks/`.

```bash
$ sudo ls -l /etc/libvirt/qemu/networks/
total 8
drwxr-xr-x 2 root root 4096 Nov  1 12:15 autostart
-rw-r--r-- 1 root root  228 Oct  6 21:13 default.xml
```

But do not edit or create a new file with your favorite text editor. libvirt gives us a tool to manage its virtual networks. After using this tool to open a file and edit, its post-save script will apply the new configurations, applying them to the appropriate tools (eg DHCP Server, iptables).

View available networks.

```bash
$ virsh net-list --all
 Name                 State      Autostart     Persistent
----------------------------------------------------------
 default              active     yes           yes
``` 

Edit a network file.

```bash
$ virsh net-edit default
```

My configuration shows:

```xml
<network>
  <name>default</name>
  <uuid>80b3425e-848b-4187-baf9-dd915e1d84c1</uuid>
  <forward mode='nat'/>
  <bridge name='virbr0' stp='on' delay='0'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
  </ip>
</network>
```

Ah-ha!  Did you realize virbr0 actually has an IP address assigned to it?

Try pinging `192.168.122.1` from an external client.

```bash
$ ping -c 3 192.168.122.1
PING 192.168.122.1 (192.168.122.1) 56(84) bytes of data.

--- 192.168.122.1 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2015ms
```

The ping request fails because this configuration has vmbr0 setup in "NAT" mode, as opposed to "route" mode - which we'll discuss later. 

Again, if you haven't already, I would suggest reading through this libvirt wiki on [Virtual Networking](http://wiki.libvirt.org/page/VirtualNetworking). It has diagrams and talks about NAT versus route modes on the virtual bridge definition.

---

To this point, we've seen the changes libvirt makes to the system during the default install.

But what about Openvswitch?  Did it make any changes?

```bash
$ sudo ovs-vsctl show
261a513a-ccdb-447d-a31b-5c6296eb102b
    ovs_version: "2.0.2"
```

No networking changes occurred during the Openvswitch installation. (Whew!)  We will configure Openvswitch later!

## Summary of interface choice configurations

What are my interfaces choices and how do I understand how to interact with them?

One fact to accept is that multiple interfaces can be assigned to use a single device definition (eg eth0) and that the single device definition knows how to manage data for that assigned interface. For example, five rows below (eth0, eth0:1, eth0:2, br1 and virbr1) map to device definition eth0.

| Interface      | Subsystem Responsible  | Maps to Device Definition (Default)               | By default, can an external device ping the IP assigned to the Interface?                                |
|:---------------|:-----------------------|:--------------------------------------------------|:---------------------------------------------------------------------------------------------------------|
| eth0           | linux                  | eth0                                              | yes                                                                                                      |
| eth0:1         | linux                  | eth0                                              | yes b/c this is an [alias interface](https://wiki.debian.org/NetworkConfiguration#Multiple_IP_addresses_on_One_Interface) |
| eth1           | linux                  | eth1                                              | yes                                                                                                      |
| wlan0          | linux                  | wlan0                                             | yes, if connected through the same WIFI router                                                           |
| br0            | linux; port = **eth0** | eth0                                              | yes b/c it uses eth0, which is a direct physical device definition                                       |
| br1            | linux; port = **none** | no mapping (isolated bridge)                      | no: needs iptables rules to explicitly map incoming data from a device to br0 or vice-versa w/ NAT       |
| virbr0         | libvirt; NAT mode      | eth0 via DNSMASQ and iptables                     | yes b/c it uses libvirt's virsh net-* commands to create bridge, DNSMASQ, DHCP Server and iptables rules |
| virbr1         | libvirt; route mode    | eth0, when user chooses route interface           | yes b/c it uses libvirt's virsh net-* commands to create a routable bridge                               |
| ovsbr0 (to-do) | openvswitch            | br0                                               | yes                                                                                                      |
| bond0 (to-do)  | linux                  | eth0, eth1                                        | yes                                                                                                      |

Another fact to accept is that not all interfaces are configured in the traditional location of `/etc/network/interfaces` but rather libvirt's virtual bridges must be edited through `virsh` even though they are saved in `/etc/libvirt/qemu/networks/` and Openvswitch commands use `ovs-vsctl`. 

The remaining examples cover common implementations for each of these Interfaces.

## eth0 (dhcp and static)

Open the linux subsystem networking `interfaces` file.

```bash
$ sudo vim /etc/network/interfaces
```

Remember that during installation, eth0 was told to use DHCP.

```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet dhcp
```

> Note: The loopback interface (`lo`) will never be changed in any of the following examples.

Let's replace DHCP with a static IP. Recall that the IP address currently assigned is `10.10.11.248`. Our new static address will be `10.10.11.50` and we'll also assign the default gateway and dns-server.

```
# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet static
   address 10.10.11.50
   netmask 255.255.255.0
   network 10.10.11.0
   gateway 10.10.11.1
   dns-nameservers 10.10.11.1
```

Reboot the system. Since we are ssh'ed into Ubuntu, Ubuntu server prevents us from [bouncing the network](http://askubuntu.com/questions/441619/how-to-successfully-restart-a-network-without-reboot-over-ssh). I would rather reboot anyways because in a production environment, we want 100% guarantee to know that when the system reboots our networking changes will all come-up okay.

```bash
$ sudo reboot now
```

ssh back into the server but using IP address `10.10.11.50`.

```bash
$ ssh USER@10.10.11.50
```

Are you logged back in? Great. Moving on to network interface aliases.

## eth0:0 (multiple IPs using aliases on a single network interface)

Debian-based systems allow us to assign [multiple IP addresses](https://wiki.debian.org/NetworkConfiguration#Multiple_IP_addresses_on_One_Interface) to a single interface.  These are called aliases and the format is INTERFACE:ALIAS-NUMBER (eg eth0:0, eth0:1, eth0:2). Aliases cannot have default gateway or dns-servers assigned to them. They use the gateway and dns-server of the default interface.

Open our `interfaces` file.

```bash
$ sudo vim /etc/network/interfaces
```

Modify. Let's turn the root back to a DHCP address, so gateway and dns are auto-discovered. Then we create aliases, even on different subnets, and let them use the same root interface.

```
# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet dhcp

# alias 0 on interface eth0, re-establishing IP 10.10.11.50
auto eth0:0
iface eth0:0 inet static
    address 10.10.11.50
    netmask 255.255.255.0

# alias 1 on interface eth0, pointing to network 192.168.77.0
auto eth0:1
iface eth0:1 inet static
    address 192.168.77.1
    netmask 255.255.255.0
```

**OR** use this syntax, which is equivelent to that listed above.

```
# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface (DHCP) w/ two alias interfaces
auto eth0
iface eth0 inet dhcp
   up   ip addr add 10.10.11.50/24 dev $IFACE label $IFACE:0
   down ip addr del 10.10.11.50/24 dev $IFACE label $IFACE:0
   up   ip addr add 192.168.77.1/24 dev $IFACE label $IFACE:1
   down ip addr del 192.168.77.1/24 dev $IFACE label $IFACE:1
```

Reboot.

```bash
$ sudo reboot now
```

ssh back into the server but using the same IP address `10.10.11.50`.

```bash
$ ssh USER@10.10.11.50
```

What does ifconfig show?

```bash
$ ifconfig
eth0      Link encap:Ethernet  HWaddr 52:54:00:87:a5:22  
          inet addr:10.10.11.248  Bcast:10.10.11.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1

eth0:0    Link encap:Ethernet  HWaddr 52:54:00:87:a5:22  
          inet addr:10.10.11.50  Bcast:10.10.11.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1

eth0:1    Link encap:Ethernet  HWaddr 52:54:00:87:a5:22  
          inet addr:192.168.77.1  Bcast:192.168.77.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0

virbr0    Link encap:Ethernet  HWaddr 6a:fe:75:50:75:8d  
          inet addr:192.168.122.1  Bcast:192.168.122.255  Mask:255.255.255.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
```

Looking good so far. Can we ping from the server to the public internet?

```bash
$ ping -c 3 www.google.com
PING www.google.com (74.125.225.114) 56(84) bytes of data.
64 bytes from ord08s08-in-f18.1e100.net (74.125.225.114): icmp_seq=1 ttl=52 time=18.4 ms
64 bytes from ord08s08-in-f18.1e100.net (74.125.225.114): icmp_seq=2 ttl=52 time=18.6 ms
64 bytes from ord08s08-in-f18.1e100.net (74.125.225.114): icmp_seq=3 ttl=52 time=17.6 ms

--- www.google.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 17.613/18.241/18.669/0.453 ms
```

Good. 

Can I ping the assigned interface and aliases from an external client?

We are already signed-in via ssh on IP address `10.10.11.50`, so we know that IP is valid.

What about `10.10.11.248`?

```bash
$ ping -c 3 10.10.11.248
PING 10.10.11.248 (10.10.11.248) 56(84) bytes of data.
64 bytes from 10.10.11.248: icmp_seq=1 ttl=64 time=0.446 ms
64 bytes from 10.10.11.248: icmp_seq=2 ttl=64 time=0.198 ms
64 bytes from 10.10.11.248: icmp_seq=3 ttl=64 time=0.370 ms

--- 10.10.11.248 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2000ms
rtt min/avg/max/mdev = 0.198/0.338/0.446/0.103 ms
```

Perfect. And `192.168.77.1`?

```bash
$ ping -c 3 192.168.77.1
PING 192.168.77.1 (192.168.77.1) 56(84) bytes of data.
From 192.168.77.4 icmp_seq=1 Destination Host Unreachable
From 192.168.77.4 icmp_seq=2 Destination Host Unreachable
From 192.168.77.4 icmp_seq=3 Destination Host Unreachable

--- 192.168.77.1 ping statistics ---
3 packets transmitted, 0 received, +3 errors, 100% packet loss, time 1999ms
pipe 3
```

Ouch. No luck. What happened?

This is a routing issue. My interface is part of the 10.10.11.0/24 subnet and not the subnet for 192.168.77.0/24. The Test Server's network interface card has knowledge of the 192.168.77.0/24 subnet but the network interface on the opposite end of the Test Server does not know of this subnet. Configure the external router's interface to recognize and route packets to the 192.168.77.0/24 subnet.

Since my external router is my Linux laptop, I add the route and then ping sucessfully.

```
$ sudo route add -net 192.168.77.0 netmask 255.255.255.0 br0
$ ping -c 3 192.168.77.1
PING 192.168.77.1 (192.168.77.1) 56(84) bytes of data.
64 bytes from 192.168.77.1: icmp_seq=1 ttl=64 time=0.238 ms
64 bytes from 192.168.77.1: icmp_seq=2 ttl=64 time=0.372 ms
64 bytes from 192.168.77.1: icmp_seq=3 ttl=64 time=0.207 ms

--- 192.168.77.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1998ms
rtt min/avg/max/mdev = 0.207/0.272/0.372/0.072 ms
```

---

In what situations would you use interface aliases?

I use aliases all the time when I take a laptop into the field and need to connect to a router or switch. Instead of redoing the `interfaces` file, I'll actually keep the defaults but run the following command.

```bash
$ sudo ifconfig eth0:2 192.168.77.2 netmask 255.255.255.0 broadcast 192.168.77.255
$ ifconfig eth0:2
eth0:2    Link encap:Ethernet  HWaddr 52:54:00:87:a5:22  
          inet addr:192.168.77.2  Bcast:192.168.77.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
```

Even though the subnet is different, I have immediate connectivity because the router or switch I'm connecting with has knowledge of the different subnet. I don't need to add routes to my routing table.

I have also seen it used to map incoming requests on the edge router to a single internal web-server host. The incoming request's public static IP ending would correspond to the internal web-server's static IP ending(eg if public IP is 4.1.1.222 then internal IP would be 192.168.77.222).  Then Apache would listen to incoming requests on that IP and server pages accordingly.

## eth1 (a second network interface)

Remember that the Test Server has two physical network ports. One was auto-assigned by dev-mapper to the `eth0` interface and the second to `eth1`. Time to configure eth1.

Open the `interfaces` file.

```bash
$ sudo vim /etc/network/interfaces
```

Add eth1 as a static IP in the same 10.10.11.0/24 network.

```
# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet dhcp
   up   ip addr add 10.10.11.50/24 dev $IFACE label $IFACE:0
   down ip addr del 10.10.11.50/24 dev $IFACE label $IFACE:0
   up   ip addr add 192.168.77.1/24 dev $IFACE label $IFACE:1
   down ip addr del 192.168.77.1/24 dev $IFACE label $IFACE:1

# Secondary network interface
auto eth1
iface eth1 inet static
   address 10.10.11.60
   netmask 255.255.255.0
   network 10.10.11.0
   gateway 10.10.11.1
   dns-nameservers 10.10.11.1
```

Reboot.

```bash
$ sudo reboot now
```

ssh back into the Test Server.

```bash
$ ssh USER@10.10.11.50
```

Now ifconfig shows the eth1 interface.

```bash
$ sudo ifconfig eth1
eth1      Link encap:Ethernet  HWaddr 52:54:00:2e:52:ed  
          inet addr:10.10.11.60  Bcast:10.10.11.255  Mask:255.255.255.0
          inet6 addr: fe80::5054:ff:fe2e:52ed/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
```

Which can also be ping'ed externally.

```bash
$ ping -c 3 10.10.11.60
PING 10.10.11.60 (10.10.11.60) 56(84) bytes of data.
64 bytes from 10.10.11.60: icmp_seq=1 ttl=64 time=0.212 ms
64 bytes from 10.10.11.60: icmp_seq=2 ttl=64 time=0.328 ms
64 bytes from 10.10.11.60: icmp_seq=3 ttl=64 time=0.199 ms

--- 10.10.11.60 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1998ms
rtt min/avg/max/mdev = 0.199/0.246/0.328/0.059 ms
```

## br0 (bridge)


