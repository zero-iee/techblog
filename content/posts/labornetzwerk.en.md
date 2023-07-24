---
title: "Setting up a lab network on Linux"
date: 2023-07-21T13:55:47+02:00
draft: false
author: "Thomas Leister"
tags: ["embedded", "network", "linux"]
---

During my work, I regularly connect to various computers and embedded devices that are accessible via an Ethernet connection. These could now be connected directly - like the development computer - to the company network...

... or you can create your own "lab network" for your devices, which is only accessible from your own laptop and over which you have full control. Advantages can be:

* Overview of the connected devices and their IP addresses.
* No exposure of the connected devices to the company network (improvement of security)
* If only Wifi is available at the development computer, the embedded devices can still be reached easily and wired.

In larger companies, access to the internal company network may also be heavily regulated, so that only unlocked devices can be used at all. With an own small laboratory network on a second network interface, this problem can be elegantly circumvented.

<!--more-->


## The plan

The lab network gets the IPv4 network "10.0.0.1/24". IP addresses are assigned between 10.0.0.10 and 10.0.0.254. The own development computer shall act as router and DNS resolver in it and has the IP address 10.0.0.1.

The network parameters on the lab interface are set statically in the network settings:

* IP address: 10.0.0.1
* Netmask: /24 (255.255.255.0)
* Gateway: 10.0.0.1

## Using Dnsmasq as DHCP and DNS server

The `dnsmasq` server is used to provide a DHCP and DNS server in the lab network. DNS is needed if an internet connection is to be used via NAT (see last section).

	sudo apt install dnsmasq

In the new configuration file `/etc/dnsmasq.d/labnet.conf` `dnsmasq` is configured:

	listen-address=10.0.0.1
	bind-interfaces
	dhcp-range=10.0.0.10,10.0.0.254,255.255.255.0,24h
	dhcp-option=option:router,10.0.0.1
	dhcp-option=option:dns-server,10.0.0.1

`listen-address` and `bind-interfaces` ensure that the supplied DNS resolver only listens on the network interface that has the IP address 10.0.0.1 assigned to it. If you omit the `bind-interfaces` option, `dnsmasq` will try to install a DNS service on the localhost interface as well, which will fail on modern Linux distributions because `systemd-resolved` is already running here. 

The new config file is still activated at the bottom of the file `/etc/dnsmasq.conf` by commenting the following line:

	conf-dir=/etc/dnsmasq.d/,*.conf

After restarting `dnsmasq`, devices that are only plugged into the lab interface will get an IP address via DHCP and should be reachable from the development computer. Which IP address a device got can be traced in the `dnsmasq` log:

	sudo journalctl -u dnsmasq -f

## Providing a (temporary) Internet connection

To allow connected devices to find a way to the Internet for software installations and updates, a source NAT can be set up on the development machine. I use a small script for this:

	#!/bin/bash

	EXT_IFACE=ens33 # ens33 = interface to public network
	INT_IFACE=ens37 # ens37 = interface to lab network
	
	if [[ "$1" == "start" ]]; then
	        echo "Starting NAT ..."
	        sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
	        sudo iptables --table nat --append POSTROUTING --out-interface $EXT_IFACE -j MASQUERADE
	        sudo iptables --append FORWARD --in-interface $INT_IFACE -j ACCEPT
	else
	        echo "Stopping NAT ..."
	        sudo iptables --table nat --delete POSTROUTING --out-interface $EXT_IFACE -j MASQUERADE
	        sudo iptables --delete FORWARD --in-interface $INT_IFACE -j ACCEPT
	fi

_(Important: Adjust interface names `ens33` or `ens37` via variables EXT_IFACE and INT_IFACE!)_

The script is stored under `~/.local/bin/natctl` and made known in the PATH:

`~/.bashrc`:

	export PATH=$PATH:~/.local/bin

The executable bit on the script is set:

	chmod u+x  ~/.local/bin/netctl

After a `source ~/.bashrc` the `natctl` script should be available. With two simple commands the internet connection can then be enabled or disabled:

	natctl start
	natctl stop

_Note: After a reboot, the `dnsmasq` server may have to be restarted, since the lab interface may not have had the right IP address for the first startup attempt and this may have failed. If newly connected devices do not get an IP address, it is worthwhile to try a restart via `sudo systemctl restart dnsmasq`._

_Note 2: The NAT function also does not survive a reboot (on purpose). After booting, the NAT must be re-enabled via `natctl start`._
