# Ubuntu20.04-Router
Ubuntu 20.04 version of "The Ars guide to building a Linux router from scratch"

Source: https://arstechnica.com/gadgets/2016/04/the-ars-guide-to-building-a-linux-router-from-scratch/

## Intro
I think [this guide from Ars][1] is the best one to understand how to setup a linux router. But the problem was that some of the steps in the guide differs from the way of doing things in Ubuntu 20.04. This document makes reference to the original guide and provide the updated steps for Ubuntu 20.04 LTS(Focal Fossa).

In the following:
* The original text appears as quotes:
  * > like this
* My comments/updates are in plain or italics
  * like this or _this_

## Steps
Read the [original guide][1] until `Configuring network interfaces` to get a clear understanding on what a router is and how it works.

### Configuring network interfaces
From [ars][1]:
>The first thing to do is figure out which one of your network interfaces is which. The simplest way to do this is to go ahead and mark them (if they aren't already marked) LAN and WAN, making sure you only have the WAN interface actually plugged in. 

Fix:
Do an `ip a` from the bash prompt 
>on your router and see which interface has an IP address—presto, that's your WAN! 
Assuming you only have two interfaces (in my case, they were p1p1 and p4p1, although your mileage will very much differ with different hardware), you now also know which interface is your LAN interface. But if you're still not sure, you can always change which port your network cable is plugged into and check for an address again.

After establishing which is which, you'll configure these network interfaces with a quick 
` sudo nano /etc/netplan/10-router.yaml`
```bash
network:
 version: 2
 renderer: networkd
 ethernets:
  p4p1:
   dhcp4: true
  p1p1:
   addresses:
    - 192.168.1.1/24
```
Then run `sudo netplan apply`. This will make the changes effective and persistent over reboot.

You can read more about `netplan` from here: https://netplan.io/examples/

### Enabling forwarding in /etc/sysctl.conf
> The first step to making Linux route is enabling forwarding between interfaces. This is really simple: `sudo nano /etc/sysctl.conf` and uncomment (remove the leading pound sign from) the line that says `net.ipv4.ip_forward=1`. Ctrl-X to exit, Y you want to save, and you're outta there.
``` bash
# Uncomment the next line to enable packet forwarding for IPv4
net.ipv4.ip_forward=1

# Uncomment the next line to enable packet forwarding for IPv6
#  Enabling this option disables Stateless Address Autoconfiguration
#  based on Router Advertisements for this host
#net.ipv6.conf.all.forwarding=1
```
>The change you made will be live after a reboot, or you can speed things up by doing a quick `sudo sysctl -p` to force the changes you made to take effect immediately.

### Creating and starting a skeleton ruleset
The [original guide][1] was using `ifupdown` but modern ubuntu installations replace this function with `networkd-dispatcher`. You can read more about it [here][2].

> Now that your PC knows it's supposed to be a router, the next step is making sure it's choosy about what it forwards and when it does so. To do that, we want to build a firewall ruleset and make sure it's applied before the network interfaces go live.

First, let's `sudo nano /etc/networkd-dispatcher/configuring.d/01iptables.sh` to create a startup script and populate it:
```bash
#!/bin/sh
/sbin/iptables-restore < /etc/networkd-dispatcher/iptables_rules
```

Now,
```
$ sudo chown root /etc/networkd-dispatcher/configuring.d/01iptables.sh
$ sudo chmod 755 /etc/networkd-dispatcher/configuring.d/01iptables.sh
```  
> This first tells the system that your script is owned by root, then the command tells the system that it's writeable only by root and readable and executable by anybody. Since our script is in the `configuring.d` directory, it will be run before the network interfaces become available, ensuring that we won't ever be online without our ruleset protecting us.

### /etc/network/iptables
This step has change since we are using a different location to store our rules:
`/etc/networkd-dispatcher/iptables_rules`

> To start out, we're going to create the bare minimum necessary for iptables-restore not to cough up a hairball. 

Create a new ruleset with
```
$ sudo iptables -t nat -P PREROUTING ACCEPT
$ sudo iptables-save > /etc/networkd-dispatcher/iptables_rules
$ sudo nano /etc/networkd-dispatcher/iptables_rules
```
_I found it easier to use `iptables-save` to generate the first file instead of starting from a blank file._

and populate it like this:

```
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]

COMMIT

*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]

# Service rules
-A INPUT -j DROP

# Forwarding rules
-A FORWARD -j DROP

COMMIT
```

> Let's take a moment and understand the basic anatomy of our ruleset before we go adding more stuff. There are two sections, *nat and *filter. Each section begins with the appropriate *declaration and ends with COMMIT. If you don't get this right—if you don't have one of the sections, or mistype its name, or fail to end it with a COMMIT statement—your ruleset won't work right. That will be a bad time.
> 
> So far, this basic skeleton ruleset doesn't actually do much of anything, but it is the framework that we'll build upon to make our router do what we expect. Understanding it now, before it gets fleshed out, will help a lot when you're ready to actually work on it.

### Enabling NAT
> At this point, we have a PC that knows how to accept a packet on one interface, and if it's destined for an address on another interface, put it there. Since we haven't actually applied our "default drop" skeleton ruleset yet, the router isn't picky about who it comes from, what it is, or who it's going to, it just does it. It also doesn't do Network Address Translation, which is the technology that lets you have a private network full of unrouteable addresses like 192.168.0.1 while still reaching public, routeable Internet addresses like 50.31.151.33. In short, we've got some work to do.
> 
> We know which port is which, so let's set them up and comment them in our iptables ruleset right away. It's also time to insert the rule that turns NAT on. 

`$ sudo nano /etc/networkd-dispatcher/iptables_rules`

```
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]

# p4p1 is WAN interface, #p1p1 is LAN interface
-A POSTROUTING -o p4p1 -j MASQUERADE

COMMIT
```
> There you have it: one is LAN, one is WAN, and you have Network Address Translation between the two. We're still not quite ready to tap into the Internet, however. You almost certainly want your router handing out IP addresses on its LAN interface, just like any router you bought from the store would.

To go ahead and apply our new NAT-enabled ruleset, do a quick 

`sudo /etc/networkd-dispatcher/configuring.d/01iptables.sh`

This will reload iptables' ruleset with whatever we've entered into `/etc/networkd-dispatcher/iptables_rules` (which, so far, boils down to "pack sand buddy, no packets for you!").

This same command: `sudo /etc/networkd-dispatcher/configuring.d/01iptables.sh` is the one we'll run whenever we've made a change to our ruleset and want to apply the new rules.

### Setting up DHCP and DNS
> Setting up your router to handle DHCP duties is easy. 
> 
> `sudo apt-get install isc-dhcp-server` 
> 
> and configure it by inserting a subnet clause into `/etc/dhcp/dhcpd.conf`
```
subnet 192.168.99.0 netmask 255.255.255.0 {
	range 192.168.99.100 192.168.99.199;
	option routers 192.168.99.1;
	option domain-name-servers 192.168.99.1;
	option broadcast-address 192.168.99.255;
}
```
You can probably figure all that out for yourself. The local network will be 192.168.99.x, the router itself will live on 192.168.99.1, the router will serve up DNS itself, and the broadcast address goes where the broadcast address always should. To apply the configurations we just made, you just 

`sudo systemctl enable --now isc-dhcp-server.service` 

Finally, we're handing out IP addresses.

We're still missing a local DNS facility, though, which our DHCP server settings explicitly promise to provide. That's even easier: 

`sudo apt-get install bind9` 

No additional configuration needed.

### Allowing DNS and DHCP client access
> At this point, we have basically everything configured, but our router is incurably paranoid. It knows how to resolve DNS queries, hand out IP addresses, and forward traffic, but it categorically refuses to actually do any of that stuff. We need to create some rules about what it's allowed to forward and what it's allowed to accept.
> 
> Open up `/etc/networkd-dispatcher/iptables_rules` in nano (or another editor of your choice), and let's get to work on the service ruleset.

```
# Service rules

# basic global accept rules - ICMP, loopback, traceroute, established all accepted
-A INPUT -s 127.0.0.0/8 -d 127.0.0.0/8 -i lo -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -m state --state ESTABLISHED -j ACCEPT

# enable traceroute rejections to get sent out
-A INPUT -p udp -m udp --dport 33434:33523 -j REJECT --reject-with icmp-port-unreachable

# DNS - accept from LAN
-A INPUT -i p1p1 -p tcp --dport 53 -j ACCEPT
-A INPUT -i p1p1 -p udp --dport 53 -j ACCEPT

# SSH - accept from LAN
-A INPUT -i p1p1 -p tcp --dport 22 -j ACCEPT

# DHCP client requests - accept from LAN
-A INPUT -i p1p1 -p udp --dport 67:68 -j ACCEPT

# drop all other inbound traffic
-A INPUT -j DROP
```
> With this configuration, we're still too paranoid to let traffic get out to the Internet, but at least now we're willing to let the machines on our LAN get IP addresses of their own and resolve DNS hostnames. We're also willing to allow pings and traceroutes (in this case, from any interface). Finally, we've allowed SSH access from the LAN to the router so that we don't have to physically plug a keyboard into it every time we want to make a change.
> 
> If you're blindly copying and pasting along with this guide, remember one thing: p1p1 was my LAN interface name, but it might not be yours. Go back and look at your notes from setting up the interfaces and make sure to get this right.

### Allowing traffic out to the Internet
>We're going to keep this one super simple: all of our local machines are allowed to get to the Internet and do whatever they want.

```
# Forwarding rules

# forward packets along established/related connections
-A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

# forward from LAN (p1p1) to WAN (p4p1)
-A FORWARD -i p1p1 -o p4p1 -j ACCEPT

# drop all other forwarded traffic
-A FORWARD -j DROP
```
>At this point, you have a credibly working router. It hands out IP addresses, it resolves DNS queries, and it passes traffic back and forth between the LAN and the Internet. Equally important, nothing on the Internet is allowed to access anything on the LAN. Still, there is one last feature to add before considering this project done.

The rest of the guide explains how to configure Port Forwarding but until here if your iptables setup is correct you can just load it by:

`sudo /etc/networkd-dispatcher/configuring.d/01iptables.sh`

> and you're off to the races.

I hope this updated guide helps!

[1]: https://arstechnica.com/gadgets/2016/04/the-ars-guide-to-building-a-linux-router-from-scratch/

[2]: https://netplan.io/faq/#use-pre-up%2C-post-up%2C-etc.-hook-scripts