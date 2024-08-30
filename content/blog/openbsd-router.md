+++
author = "cr4sh0v3rrid3"
title = "OpenBSD router in a hurry!"
date = "2024-08-30"
description = "Setting up a new OpenBSD router in a hurry!"
tags = [
    "OpenBSD",
]
+++

## How I got myself into this situation
My old router decided to migrate into the eternal life as e-waste the other day. It completely died; not even showing signs of life on the console port! For over 8 years it had been serving internet for me and my wife more or less 24/7, so it probably felt it was time to retire after a long and prosperous life of packet shuffling. 

The, now dead, router was a PC Engines APU4C4, which I bought from teklager.se. For the duration of its life it ran pfSense, which I had lots of issues with the entire time. If it weren't for the fact that downtime would affect my wife as well, I would have installed something else a long time ago. But I digress.

I was luck this time since the router died while my wife was gone on a work related trip. That meant that I had a bit over one day to fix something new. Now, the challenge in this case was that my network topology is rather complicated compared to the "normal" networks found in the average home. I have taken the time to segment the network, by creating segments for WLAN, the servers, IoT and a subnet for even less trusted devices than the IoT subnet... As the wise and experienced network administrator I am I had of course not stored a backup anywhere accessible!

So, figuring out what VLAN IDs I had used became a rather annoying challenge. Luckily I had most of the information available on a old Cisco switch I hadn't gotten around to throw away.
## The temporary solution
I found an old Lenovo T480s laying around that I ended up installing OpenBSD on. Since that machine only had one network port I had to use a USB-C hub with an additional port. This part, luckily, worked right out of the box without any issues. 

To get this to work I needed:
- A egress interface, axen0, connected to the cable modem for internet routing
- A local interface, em0, where all the VLANs live
	- VLAN 120 - the server subnet
	- VLAN 130 - the WiFi subnet
	- VLAN 140 - the IoT subnet, used for both wired and wireless clients
	- VLAN 150 - the alarm system; completely isolated from everything else

This all boils down to:
- Configuring all the interfaces `/etc/hostname.[axen0|em0|vlan120|vlan130|vlan140|vlan150]`
	- Only the egress interface would be configured with autconf to recieve it's IP, route etc from my ISP
	- All the other interfaces will be have a static configuration
- Configure the dhcpd service to issue leases to clients in the network
- Configure pf with some sane rules
	- VLAN 120 should be allowed to talk to everything
	- VLAN 130 should be allowed to talk to everything
	- VLAN 140 should only be allowed to talk to specific services on VLAN 120
	- VLAN 150 should not be allowed to talk to anything
- Bonus: after getting everything up and running we want IPv6 support!

/etc/hostname.axen0
```txt
inet autoconf
```

/etc/hostname.em0
```txt
inet 10.0.0.1 255.255.255.0
up
```

/etc/hostname.vlan120
```txt
vlan 120 vlandev em0 
inet 192.168.1.1 255.255.255.0
```

/etc/hostname.vlan130
```txt
vlan 130 vlandev em0
inet 10.0.1.1 255.255.255.0
```

/etc/hostname.vlan140
```txt
vlan 140 vlandev em0
inet 10.0.4.1 255.255.255.0
```

/etc/hostname.vlan150
```txt
vlan 150 vlandev em0
inet 10.0.5.1 255.255.255.0
```

Then to apply these changes, we issue the following command:
```
$ sh /etc/netstart
```

Then /etc/dhcpd.conf
```txt
# Globals
option domain-name "sigkill.no";
option domain-name-servers 192.168.1.2;
default-lease-time 600;
max-lease-time 7200;

# Server subnet
subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.20 192.168.1.254;
    option routers 192.168.1.1;
    option subnet-mask 255.255.255.0;
    option broadcast-address 192.168.1.255;
}

# VLAN 130
subnet 10.0.1.0 netmask 255.255.255.0 {
    range 10.0.1.20 10.0.1.254;
    option routers 10.0.1.1;
    option subnet-mask 255.255.255.0;
    option broadcast-address 10.0.1.255;
}

# VLAN 140
subnet 10.0.4.0 netmask 255.255.255.0 {
    range 10.0.4.40 10.0.4.254;
    option routers 10.0.4.1;
    option subnet-mask 255.255.255.0;
    option broadcast-address 10.0.4.255;
    host jarvis {
        fixed-address 10.0.4.27;
        hardware ethernet dc:a6:32:77:3c:0d;
    }
}

# VLAN 150
subnet 10.0.5.0 netmask 255.255.255.0 {
    range 10.0.5.10 10.0.5.254;
    option routers 10.0.5.1;
    option subnet-mask 255.255.255.0;
    option broadcast-address 10.0.5.255;
}
```

Then installing dhcpcd for IPv6:
```
$ pkg_add dhcpcd
```

 To have IPv6 working we need /etc/dhcpcd.conf
```txt
...
allowinterfaces axen0 em0 vlan120 vlan130 vlan140 vlan150

interface axen0
  ipv6rs
  ia_na 1
  ia_pd 2 em0/1 vlan120/2 vlan130/3 vlan140/4 vlan150/5
```

aaaaand /etc/rad.conf
```txt
interface em0
interface vlan120
interface vlan130
interface vlan140
interface vlan150
```

After configuring dhcpd, dhcpcd and rad we have to enable and start the services:
```
$ rcctl enable dhcpd; rcctl start dhcpd
$ rcctl enable dhcpcd; rcctl start dhcpcd
$ rcctl enable rad; rcctl start rad
```

We also have to enable some kernel options
```txt
net.inet.ip.forwarding=1
net.inet6.ip6.forwarding=1
net.inet.ip.mforwarding=1
# Since we are using a laptop we want to disable sleep when lid is closed...
machdep.lidaction=0
```

Which we can make have apply right away without the need for a reboot:
```
$ sysctl net.inet.ip.forwarding=1
$ sysctl net.inet6.ip6.forwarding=1
$ sysctl net.inet.ip.mforwarding=1
$ sysctl machdep.lidaction=0
```

The last configuration we have to do is /etc/pf.conf
```txt
# Tables
table <lan> { 10.0.0.0/24 }
table <vlan120> { 192.168.1.0/24 }
table <vlan130> { 10.0.1.0/24 }
table <vlan140> { 10.0.4.0/24 }
table <vlan150> { 10.0.5.0/24 }

table <martians> { 0.0.0.0/8 127.0.0.0/8 169.254.0.0/16 172.16.0.0/12 192.0.0.0/24 192.0.2.0/24 224.0.0.0/3 \
                   192.168.0.0/16 198.18.0.0/15 198.51.100.0/24 203.0.113.0/24 \
                   ::/128 ::/96 ::1/128 ::ffff:0:0/96 100::/64 2001:10::/28 2001:2::/48 2001:db8::/32 3ffe::/16 fec0::/10 fc00::/7 }

icmp_types = "{ echoreq, unreach }"

# Options
set skip on lo                # Don't filter on loopback interface
set block-policy drop
set loginterface egress
set optimization aggressive
set skip on lo

block all

match in all scrub (no-df random-id)
match out on egress inet from !(egress:network) to any nat-to (egress:0)
antispoof for egress

pass out inet
pass out inet6
pass in on { lan vlan120 vlan130 vlan140 vlan150 } inet
pass in on { lan vlan120 vlan130 vlan140 vlan150 } inet6

block in on egress from any to <martians>

# Block all other traffic from VLAN 140 to VLAN 120
block in log on vlan140 from vlan140:network to vlan120:network

pass out on egress inet proto udp to port 33433:33626
pass inet proto icmp icmp-type $icmp_types

pass in on egress inet6 proto udp \
  from fe80::/10 port dhcpv6-server \
  to fe80::/10 port dhcpv6-client \
  no state

pass out on egress inet6 proto udp from any to any port 33433 >< 33626 keep state
pass inet6 proto icmp6 all

# For the trusted and work LAN's, allow devices to talk with the other devices in their respective network.
pass on vlan120 from vlan120:network to vlan120:network
pass on vlan130 from vlan130:network to vlan130:network
pass on vlan140 from vlan140:network to vlan140:network

pass in log on egress proto tcp from any to any port 22 rdr-to 192.168.1.2
pass in log on egress proto udp from any to any port 51820 rdr-to 192.168.1.2

# Allow multicast networking
pass on { vlan120 vlan130 vlan140 } proto igmp all allow-opts
pass on { vlan120 vlan130 vlan140 } inet proto udp from any to 224.0.0.0/4
pass on { vlan120 vlan130 vlan140 } inet6 proto udp from any to ff00::/8

# Allow mDNS
pass inet proto udp from any to 224.0.0.251 port { mdns } allow-opts
pass inet6 proto udp from any to ff02::fb port { mdns } allow-opts

# Allow from IOT vlan to server
pass in on vlan140 proto tcp from vlan140:network to 192.168.1.2 port 8096
pass in on vlan140 proto udp from vlan140:network to 192.168.1.2 port 53
```

Which we have to apply by issuing the following command:
```
$ pfctl -f /etc/pf.conf
```

At this point everything *should* just work.

I would highly recommend to read the extremeley good man pages for 
[ifconfig(8)](https://man.openbsd.org/ifconfig),
[dhcpd(8)](https://man.openbsd.org/dhcpd),
[rad.(8)](https://man.openbsd.org/rad)
and [pfctl(8)](https://man.openbsd.org/pfctl.8)

It would probably also be a good idea to have a look at how to configure those
things:
[hostname.if(5)](https://man.openbsd.org/hostname.if),
[dhcpd.conf(5)](https://man.openbsd.org/dhcpd.conf),
[rad.conf(5)](https://man.openbsd.org/rad.conf)
and [pf.conf(5)](https://man.openbsd.org/pf.conf.5)

## Making sure we save config files somewhere
To keep my dotfiles saved on git I've settled on the simple solution of just having a bare git repository with `$HOME` as the working tree. I found this awesome way of keeping dotfiles over at a really great [atlassian blogpost](https://www.atlassian.com/git/tutorials/dotfiles). This same system can be used to save the configuration files for the router, just using `/` as our working tree:

```
$ git init --bare /.cfg
$ alias config='/usr/bin/git --git-dir=/.cfg/ --work-tree=/'
# To make the alias stick:
$ echo "alias config='/usr/bin/git --git-dir=/.cfg/ --work-tree=/'" >> /root/.profile
$ config add /etc/hostname.* /etc/dhcpd.conf /etc/dhcpcd.conf /etc/rad.conf /etc/pf.conf /etc/sysctl.conf
$ config commit -m "feat: adding all config files"
$ config push
```

## The permanent solution
To replace the PC Engines APU4 router I looked for something with the following requirements:
- Passively cooled
- Low power consumption
- Still supported
- At least two network interfaces
- Open source BIOS, preferably Coreboot
That last requirement really narrows the choice down, as there is, unfortunately, a lack of vendors willing to use open source BIOSes (ADD CONTEXT?).

I ended up buying a [Byte from Star Labs](https://starlabs.systems/pages/byte) which checked all of those boxes.

To apply the config files from this repo we created previously to our new router:

Install needed packages:
```
$ pkg_add vim dhcpcd git
```

Configure the working tree:
```
$ alias config='/usr/bin/git --git-dir=/.cfg/ --work-tree=/'
$ echo "alias config='/usr/bin/git --git-dir=/.cfg/ --work-tree=/'" >> /root/.profile
$ git clone --bare git@git.sr.ht:~cr4sh0v3rrid3/openbsd-router /.cfg
$ config checkout
```

At this point be want rename some of the `/etc/hostname.` files to use the correct network interfaces, as well as updating all the vlan interfaces with the same changes. So we replace `axen0` with `re0` and `em0` with `rge0` 

Then you have to enable some services
```
$ rcctl enable dhcpd
$ rcctl enable dhcpcd
$ rcctl enable rad
```

To end it of just reboot the machine and plug in the correct cables to the correct
interfaces.

## Conclusion
So I finally got myself around to setting up a OpenBSD router! And now that I've used this setup for about a week I'm extremely happy and satisfied.

All the configuration files mentioned and used in this post can be found over at  my [git.sr.ht](https://git.sr.ht/~cr4sh0v3rrid3/openbsd-router) repo!

