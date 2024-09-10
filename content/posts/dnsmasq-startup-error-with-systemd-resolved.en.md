---
title: "Dnsmasq does not start because port 53 is busy"
date: 2024-09-06T17:33:54+02:00
draft: false
author: "Thomas Leister"
tags: ["dnsmasq", "systemd-resolved", "systemd", "dns", "linux", "ubuntu"]
---

<!--more-->

After installing Dnsmasq, the service started unsurprisingly with an error message:

```
$ sudo systemctl status dnsmasq
Sep 06 14:32:22 office-gateway dnsmasq[830]: failed to create listening socket for port 53: Address already in use
Sep 06 14:32:22 office-gateway dnsmasq[830]: FAILED to start up
Sep 06 14:32:22 office-gateway systemd[1]: dnsmasq.service: Control process exited, code=exited, status=2/INVALIDARGUMENT
Sep 06 14:32:22 office-gateway systemd[1]: dnsmasq.service: Failed with result 'exit-code'.
Sep 06 14:32:22 office-gateway systemd[1]: Failed to start dnsmasq.service - dnsmasq - A lightweight DHCP and caching DNS server.
```

The cause was quickly clear: Dnsmasq is apparently trying to spread itself on the local loopback interface `lo` in order to offer a DNS resolver on port 53. The problem is that this port is already occupied by `systemd-resolved`. We thought the problem could be solved simply by instructing Dnsmasq to listen only to the lab network interface. This can be set in the Dnsmasq configuration as follows: 


```
interface=enp0s31f6
```

Alternatively, all interfaces _except_ a specific one can be listened to: 

```
except-interface=lo
```

However, even after this adjustment, Dnsmasq gave us the above error message: Port 53 was already blocked. 

After some research, we came across the following Dnsmasq parameters:

> -z, --bind-interfaces
    On systems which support it, dnsmasq binds the wildcard address, even when it is listening on only some interfaces. It then discards requests that it shouldn't reply to. This has the advantage of working even when interfaces come and go and change address. This option forces dnsmasq to really bind only the interfaces it is listening on. About the only time when this is useful is when running another nameserver (or another instance of dnsmasq) on the same machine. Setting this option also enables multiple instances of dnsmasq which provide DHCP service to run in the same machine. 

_https://linux.die.net/man/8/dnsmasq_

... and that explained our problem: 

By default, Dnsmasq also binds _all_ interfaces (regardless of whether `interface=` or `except-interface=` are used!) and only ignores requests if requests are received on interfaces for which exceptions apply. Our `interface=` setting was therefore effective in a different way than we thought: Only requests on the `lo` interface were ignored. Dnsmasq still requested control over `lo` port 53 (through a “bind”).

To resolve the conflict, we simply added a `bind-interfaces` to our settings: 

```
bind-interfaces
interface=enp0s31f6
```

... and could now run Dnsmasq and `systemd-resolved` at the same time. 
