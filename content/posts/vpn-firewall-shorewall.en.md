---
title: "Keeping the Wireguard VPN firewall clear with Shorewall"
date: 2023-12-04T08:41:52+01:00
draft: false
author: "Thomas Leister"
tags: ["wireguard", "vpn", "shorewall", "firewall", "iptables"]
---


In our [previous article](/posts/multi-tenant-wireguard-vpn-server/) we introduced the iptables firewall for our Wireguard VPN server. The firewall regulates which traffic is permitted between the individual customer VPNs and the management VPN and prevents access that poses a security risk. 

Although it is possible to manage these rules using the iptables command line tools, it quickly becomes confusing and difficult to understand, especially for outsiders. We have therefore tested the firewall configuration using the "[Shorewall](https://shorewall.org/)" tool and found it to be suitable.

<!--more-->

Shorewall is a tool that reads simple text files with firewall rules according to a predefined format, validates them and converts them into iptables rules. It is therefore only a front end that operates iptables and not an independent firewall in the strict sense. 

The static configuration in individual semantically separated text files makes it easier to maintain an overview. Iptables rules can also be persisted in text files via netfilter-persistent (or iptables-persistent), but the syntax is difficult to understand at a glance. This may be sufficient for smaller setups - but we are planning to set up some VPN networks with individual authorisations, among other things. With Shorewall as an iptables configurator, we can keep a better overview of the firewall rules and avoid errors. 

With Shorewall, the firewall configuration is distributed across several files in the configuration directory `/etc/shorewall`:

- `/etc/shorewall/zones`: Defines the firewall zones that can later be used in the rule sets.
- `/etc/shorewall/interfaces`: Defines which network interfaces are to be used with Shorewall and which zone they belong to.
- `/etc/shorewall/policy`: Defines the default policies between zones. Is traffic allowed to pass between zones or not?
- `/etc/shorewall/rules`: The fine-grained rules are defined here: For example, if a `REJECT` policy was previously defined between zones, port/protocol-based exceptions can be defined here. 


Below we take a look at our configuration:

`zones`:

```
fw              firewall
wan             ipv4
customer1-vpn   ipv4
customer1-vpn   ipv4
mastervpn       ipv4

```

We define zone `fw` (alias: `$FW`) as the firewall's own zone and define further zones for our WAN interface and our VPN networks.


`interfaces`:

```
wan             eth0            detect                  dhcp,routefilter,tcpflags

customer1-vpn   customer1-vpn   detect                  routeback
customer2-vpn   customer2-vpn   detect                  routeback
mastervpn       mastervpn       detect                  routeback
```


The network interface and zone are linked here - the zone in the first column, the assigned network interface in the second column. The `routeback` option is particularly important for the VPN zones, which ensures that traffic arriving at a VPN interface can also leave it again directly, as is the rule with a VPN interface when clients communicate with each other. This client-to-client communication will be deactivated again later, but we want to retain the option of enabling it for individual VPN clients. 

`policy`:

```
# Source        # Dest  # Policy
wan             all     REJECT
$FW             all     ACCEPT

mastervpn       all     ACCEPT
customer1-vpn   all     REJECT
customer2-vpn   all     REJECT

```


Incoming traffic on the WAN should always be blocked first. Individual exceptions - for example for the VPN server ports - are made later in the `rules` file. 

The firewall itself has no restrictions and can address other zones or interfaces as required. 

Our `mastervpn` may also communicate without restrictions. This is the admin interface from which our developers establish connections to various VPN clients. 

We take a more restrictive approach with the two customer VPNs `customer1-vpn` and `customer2-vpn`: These are locked in using `REJECT` and are not allowed to communicate with anyone by default. 


`rules`: The exceptions for the policies just explained now follow:

```
# Policy        # Source        # Dest  # Prot  # Port

# Allow access to VPN ports
ACCEPT          wan             $FW     UDP     51821,51822,51823

# Allow SSH from internet
SSH(ACCEPT)     wan             $FW

# Allow pings from all VPNs to their Server VPN interface
Ping(ACCEPT)    customer1-vpn        $FW:10.2.1.1
Ping(ACCEPT)    customer2-vpn        $FW:10.3.0.1
Ping(ACCEPT)    mastervpn            $FW:10.4.0.1

# Explicitly allow inter-client connections on some VPN devices (legacy "admin" devices)
ACCEPT          customer1-vpn:10.2.1.10    customer1-vpn      # device 1
ACCEPT          customer1-vpn:10.2.1.11    customer1-vpn      # device 2
ACCEPT          customer1-vpn:10.2.1.12    customer1-vpn      # device 3

# Prevent inter-client connections on VPNs for all devices that have not explicitly been allowed in the section before
REJECT          customer1-vpn        customer1-vpn:10.2.1.0/24
REJECT          customer2-vpn        customer2-vpn:10.3.0.0/16
```


Firstly, access to the WAN ports `51821,51822,51823` is permitted so that Wireguard VPN clients can connect to the VPN server. We also connect to the VPN server via SSH - this should also remain permitted. In this case, we use the `SSH(ACCEPT)` macro from Shorewall so that we do not have to define the port and protocol separately. 

Pings from VPN clients to their respective server-side interface and its IP addresses should be allowed in all cases. We usually ping the server interface to determine the functionality of a VPN connection. 

In the next section, client-to-client communication is explicitly enabled for individual VPN clients of the `customer1-vpn` network. 

... For all other devices, this type of communication is prevented in the following section using `REJECT`. Consequently, they can only send back response packets (implicit rule in Shorewall) and cannot initiate connections of their own accord. 

IP forwarding is already activated on our server. If you have not yet activated the setting `net.ipv4.ip_forward` in /etc/sysctl.conf, you can also set `IP_FORWARDING` to `On` in `/etc/shorewall/shorewall.conf` instead. 


Finally, we took 'netfilter-persistent' out of the boat:

	systemctl disable netfilter-persistent

... and emptied the iptables rules: 

	iptables -F

... to be able to test the new Shorewall rules:

	shorewall start

After everything worked as hoped, Shorewall was added to the autostart:

	systemctl enable shorewall
