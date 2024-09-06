---
title: "Dnsmasq startet nicht wegen belegtem Port 53"
date: 2024-09-06T17:33:54+02:00
draft: false
author: "Thomas Leister"
tags: ["dnsmasq", "systemd"]
---

Vielleicht ist der ein oder andere auch schon einmal in das Problem gelaufen, als Dnsmasq als separater DNS/DHCP Server auf einem Host mit einem extra Netzwerkinterface eingesetzt werden sollte: 

```
$ sudo systemctl status dnsmasq

[...]
Sep 06 14:32:22 office-gateway dnsmasq[830]: failed to create listening socket for port 53: Address already in use
Sep 06 14:32:22 office-gateway dnsmasq[830]: FAILED to start up
Sep 06 14:32:22 office-gateway systemd[1]: dnsmasq.service: Control process exited, code=exited, status=2/INVALIDARGUMENT
Sep 06 14:32:22 office-gateway systemd[1]: dnsmasq.service: Failed with result 'exit-code'.
Sep 06 14:32:22 office-gateway systemd[1]: Failed to start dnsmasq.service - dnsmasq - A lightweight DHCP and caching DNS server.
[...]
```

Die Ursache war uns - dachten wir - schnell klar, denn auf dem fraglichen Ubuntu Server System befindet sich an Werk bereits der lokale `systemd-resolved` Resolver, der auf Port 53 bereits hervorragende Leiste leistet. Der Dnsmasq-Service, der neben DHCP auch einen DNS-Service auf Port 53 anbieten will, kann den Port so nicht mehr an sich reißen. 

Wir dachten, durch eine Ausnahme bei den Netzwerkinterfaces könnten wir das Problem umgehen, denn über `except-interface` können für Dnsmasq Interfaces angegeben werden, die ignoriert werden sollen. Datei `/etc/dnsmasq.conf`:

```
except-interface=lo
```

doch das hat nichts genützt. Das Problem bestand noch immer.

Schließlich sind wir in `/etc/defaults/dnsmasq` auf eine Option aufmerksam geworden, die Dnsmasq veranlasst, sämtliche lokale Schnittstellen außer Acht zu lassen: 

```
# If the resolvconf package is installed, dnsmasq will tell resolvconf
# to use dnsmasq under 127.0.0.1 as the system's default resolver.
# Uncommenting this line inhibits this behaviour.
DNSMASQ_EXCEPT="lo"
```

Die Option muss - wie dargestellt - einkommentiert werden, damit sie aktiv wird. Nach einem Neustart des  `dnsmasq` Services kommen sich die beiden DNS-Services nicht mehr in die Quere. 
Wir haben uns zusätzlich dazu entschieden, Dnsmasq explizit nur auf dem gewünschten Interface zu aktivieren. Datei `/etc/dnsmasq.conf`:

```
interface=enp0s31f6
```

_(diese Maßnahme alleine führte übrigens auch nicht zum Erfolg - DNSMASQ_EXCEPT="lo" scheint dennoch notwendig zu sein.)_


