---
title: "Belegter Port 53 verhindert Start von Dnsmasq"
date: 2024-09-06T17:33:54+02:00
draft: false
author: "Thomas Leister"
tags: ["dnsmasq", "systemd-resolved", "systemd", "dns", "linux", "ubuntu"]
---

Auf einem Gateway-Host, der ein spezielles Labornetz innerhalb unseres Firmennetzwerks aufspannt, sollte ein Dnsmasq-Server zum Einsatz kommen, um in diesem neuen Labornetzwerk nicht nur DHCP bereitzustellen, sondern auch einen DNS-Resolver. Gleichzeitig sollte der auf dem Ubuntu Server vorinstallierte `systemd-resolved` Service nicht angetastet werden, da wir ihn später noch als [Split-Horizon DNS Resolver](https://blogs.gnome.org/mcatanzaro/2020/12/17/understanding-systemd-resolved-split-dns-and-vpn-configuration/) benötigen werden. 

<!--more-->

Nach der Installation von Dnsmasq startete der Dienst wenig überraschend mit einer Fehlermeldung:

```
$ sudo systemctl status dnsmasq
Sep 06 14:32:22 office-gateway dnsmasq[830]: failed to create listening socket for port 53: Address already in use
Sep 06 14:32:22 office-gateway dnsmasq[830]: FAILED to start up
Sep 06 14:32:22 office-gateway systemd[1]: dnsmasq.service: Control process exited, code=exited, status=2/INVALIDARGUMENT
Sep 06 14:32:22 office-gateway systemd[1]: dnsmasq.service: Failed with result 'exit-code'.
Sep 06 14:32:22 office-gateway systemd[1]: Failed to start dnsmasq.service - dnsmasq - A lightweight DHCP and caching DNS server.
```

Die Ursache war schnell klar: Offenbar versucht Dnsmasq, sich auf der lokalen Loopback-Schnittstelle `lo` breit zu machen, um dort auf Port 53 einen DNS-Resolver anzubieten. Das Problem liegt nun darin, dass sich dieser Port bereits durch `systemd-resolved` belegt ist. Das Problem lässt sich - dachten wir - einfach lösen, indem wir Dnsmasq anweisen, nur auf das Labornetz-Interface zu horchen. In der Dnsmasq-Konfiguration kann das wie folgt eingestellt werden: 

```
interface=enp0s31f6
```

Alternativ kann auch auf alle Interfaces _außer_ ein bestimmtes gehorcht werden: 

```
except-interface=lo
```

Doch auch nach dieser Anpassung warf Dnsmasq uns die obenstehende Fehlermeldung entgegen: Port 53 sei bereits blockiert. 

Nach einiger Recherche stießen wir auf folgenden Dnsmasq Parameter:

> -z, --bind-interfaces
    On systems which support it, dnsmasq binds the wildcard address, even when it is listening on only some interfaces. It then discards requests that it shouldn't reply to. This has the advantage of working even when interfaces come and go and change address. This option forces dnsmasq to really bind only the interfaces it is listening on. About the only time when this is useful is when running another nameserver (or another instance of dnsmasq) on the same machine. Setting this option also enables multiple instances of dnsmasq which provide DHCP service to run in the same machine. 

_https://linux.die.net/man/8/dnsmasq_

... und das erklärte unser Problem: 

Standardmäßig bindet Dnsmasq auch _alle_ Interfaces (egal, ob nun `interface=` oder `except-interface=` genutzt werden!) und ignoriert Anfragen lediglich, wenn Anfragen auf Schnittstellen eingehen, für die Ausnahmen gelten. Unsere `interface=` Einstellung war also auf eine andere Art und Weise wirksam, als wir dachten: Es wurden lediglich Anfragen am `lo` Interface ignoriert. Dnsmasq forderte dennoch die Kontrolle über `lo` Port 53 (durch einen "bind").

Um den Konflikt zu lösen, haben wir in unsere Einstellungen also schlicht ein `bind-interfaces` mit aufgenommen: 

```
bind-interfaces
interface=enp0s31f6
```

... und konnten nun Dnsmasq neben `systemd-resolved` starten.
