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

The laboratory network at the network interface `ens37` gets the IPv4 network `10.0.0.1/24`. IP addresses are assigned between 10.0.0.10 and 10.0.0.254. The own development computer is to act in it as router and DNS resolver and has the IP address `10.0.0.1`.

The network parameters on the lab interface are set statically in the network settings:

* IP address: 10.0.0.1
* Netmask: /24 (255.255.255.0)
* Gateway: 10.0.0.1

## Using Dnsmasq as DHCP and DNS server

The `dnsmasq` server is used to provide a DHCP and DNS server in the laboratory network. DNS is needed if an internet connection is to be used via NAT (see last section).

	sudo apt install dnsmasq

In the new configuration file `/etc/dnsmasq.d/labnet.conf` dnsmasq is configured:

	# interface to use
	interface=ens37
	
	# DHCP settings
	dhcp-authoritative
	dhcp-range=10.0.0.100,10.0.0.250,255.255.255.0,12h
	dhcp-option=option:router,10.0.0.1
	dhcp-option=option:dns-server,10.0.0.1
	
	# Always assign the same IP addresses to these hosts
	dhcp-host=10:00:00:00:00:01,10.0.0.100
	dhcp-host=10:00:00:00:00:01,10.0.0.101
	
	# Upstream DNS resolver
	server=127.0.0.53
	
	# Local TLD
	domain=lab
	local=/lab/
	expand-hosts
	
	# Static DNS hostnames in labnet
	address=/mydevice-altname.com/10.0.0.101

`interface=ens37` makes the DNS resolver listen only on the network interface `ens37`. If you omit the setting, `dnsmasq` tries to start a DNS service on the localhost interface as well, which fails on modern Linux distributions, because `systemd-resolved` is already running here. 

Subsequently, the DHCP range is defined from which IP addresses are to be distributed to devices in the laboratory network (`10.0.0.100 - 10.0.0.250`). This guarantees that an IP address remains valid for at least 12 hours.

The `dhcp-option` settings are used to tell the terminated devices under which IP addresses the default gateway and DNS resolver are located. In both cases, this is the host that runs dnsmasq.

With `dhcp-host` IP addresses for individual network interfaces or devices can be "fixed". Thus the device with the MAC address `10:00:00:00:0` is always assigned the IP address `10.0.0.101`.

If the devices connected to the laboratory network are to have Internet access (see section _"Providing a (temporary) Internet connection"_), an upstream DNS server must be specified for the dnsmasq instance. Dnsmasq cannot resolve global domain names such as google.com itself and instead falls back on the DNS server stored here. Here it is set to the IP address of the systemd-resolved service that comes with most modern Linux distributions.

The three following parameters (`domain=` ff.) define which top-level domain should be used within the lab network. This is important to be able to tell systemd-resolved later which DNS resolver (namely dnsmasq!) should be used for the lab network. 
If you want to manipulate public DNS records or simulate your own virtual ones, you can set one or more `address=` parameters. The domain name mentioned in it will then be statically converted to the IP address set behind it (and the upstream DNS server will be bypassed). 


The new config file is still activated at the bottom of the `/etc/dnsmasq.conf` file by commenting the following line:

	conf-dir=/etc/dnsmasq.d/,*.conf

Finally, systemd-resolved is told to automatically resolve domain names ending in `.lab` using the dnsmasq server instead of any other possibly public DNS server (which of course would not know the names on the lab network):

	sudo resolvectl domain ens37 lab
	sudo resolvectl dns ens37 10.0.0.1


After restarting `dnsmasq`, devices that are only plugged into the lab interface will pull an IP address via DHCP and should be reachable from the development computer. Which IP address a device got can be traced in the `dnsmasq` log:

	sudo journalctl -u dnsmasq -f

If a device with the hostname `mydevice` reports to the DHCP server, it can be pinged using the hostname `mydevice.lab` for example, or an SSH session can be opened as follows:

	ssh user@mydevice.lab



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

	chmod u+x  ~/.local/bin/natctl

After a `source ~/.bashrc` the `natctl` script should be available. With two simple commands the internet connection can then be enabled or disabled:

	natctl start
	natctl stop

_Note: After a reboot, the `dnsmasq` server may have to be restarted, since the lab interface may not have had the right IP address for the first startup attempt and this may have failed. If newly connected devices do not get an IP address, it is worthwhile to try a restart via `sudo systemctl restart dnsmasq`._

_Note 2: The NAT function also does not survive a reboot (on purpose). After booting, the NAT must be re-enabled via `natctl start`._
