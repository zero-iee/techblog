---
title: "Dnsmasq does not start because port 53 is busy"
date: 2024-09-06T17:33:54+02:00
draft: true
author: "Thomas Leister"
tags: ["dnsmasq", "systemd"]
---

Perhaps some of you have already run into this problem when Dnsmasq was to be used as a separate DNS/DHCP server on a host with an extra network interface: 

```
$ sudo systemctl status dnsmasq
Sep 06 14:32:22 office-gateway dnsmasq[830]: failed to create listening socket for port 53: Address already in use
Sep 06 14:32:22 office-gateway dnsmasq[830]: FAILED to start up
Sep 06 14:32:22 office-gateway systemd[1]: dnsmasq.service: Control process exited, code=exited, status=2/INVALIDARGUMENT
Sep 06 14:32:22 office-gateway systemd[1]: dnsmasq.service: Failed with result 'exit-code'.
Sep 06 14:32:22 office-gateway systemd[1]: Failed to start dnsmasq.service - dnsmasq - A lightweight DHCP and caching DNS server.

```

The cause was - we thought - quickly clear to us, because the Ubuntu server system in question is already equipped with the local `systemd-resolved` resolver, which already performs excellently on port 53. The Dnsmasq service, which in addition to DHCP also wants to offer a DNS service on port 53, can no longer hijack the port. 

We thought that we could avoid the problem by making an exception for the network interfaces, because interfaces that should be ignored can be specified for Dnsmasq via `except-interface`. File `/etc/dnsmasq.conf`:

```
except-interface=lo
```

But that didn't help. The problem still persisted.

Finally, we found an option in `/etc/defaults/dnsmasq` that causes Dnsmasq to ignore all local interfaces: 

```
# If the resolvconf package is installed, dnsmasq will tell resolvconf
# to use dnsmasq under 127.0.0.1 as the system's default resolver.
# Uncommenting this line inhibits this behaviour.
DNSMASQ_EXCEPT="lo"
```

The option must be commented in - as shown - for it to become active. After restarting the `dnsmasq` service, the two DNS services will no longer interfere with each other. 
We have also decided to explicitly activate Dnsmasq only on the desired interface. File `/etc/dnsmasq.conf`:

```
interface=enp0s31f6
```

_(this measure alone did not lead to success - `DNSMASQ_EXCEPT="lo"` seems to be necessary!)_


