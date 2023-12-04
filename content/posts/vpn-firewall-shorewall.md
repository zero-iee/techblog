---
title: "Die Wireguard VPN Firewall mit Shorewall übersichtlich halten"
date: 2023-12-04T08:41:52+01:00
draft: false
author: "Thomas Leister"
tags: ["wireguard", "vpn", "shorewall", "firewall", "iptables"]
---


In unserem [vorherigen Artikel](/posts/multi-tenant-wireguard-vpn-server/) haben wir die iptables-Firewall für unseren Wireguard VPN-Server vorgestellt. Die Firewall regelt, welcher Traffic zwischen den einzelnen Kunden-VPNs und dem Management-VPN erlaubt ist und verhindert Zugriffe, die ein Sicherheitsrisiko darstellen. 

Die Verwaltung dieser Regeln mittels der iptables Command Line Tools ist zwar möglich, allerdings relativ schnell unübersichtlich und insbesondere für Außenstehende schwer zu erfassen. Wir haben deshalb die Firewallkonfiguration über das Tool "[Shorewall](https://shorewall.org/)" erprobt und für geeignet befunden.

<!--more-->

Shorewall ist ein Tool, das einfache Textdateien mit Firewallregeln nach einem vordefinierten Format einliest, validiert und in iptables-Regeln umsetzt. Es ist also nur ein Frontend, das iptables bedient und keine eigenständige Firewall im engeren Sinne. 

Durch die statische Konfiguration in einzelnen semantisch getrennten Textdateien fällt es leichter, den Überblick zu behalten. Iptables-Regeln können zwar via netfilter-persistent (bzw. iptables-persistent) ebenfalls in Textdateien persistiert werden, doch die Syntax ist beim Überfliegen nur schwer verständlich. Für kleinere Setups mag das ausreichen - doch wir planen einige VPN-Netze u.A. mit individuellen Berechtigungen aufzuspannen. Mit Shorewall als iptables-Konfigurator können wir die Übersicht über die Firewallregeln besser behalten und Fehler vermeiden. 

Die Firewallkonfiguration verteilt sich bei Shorewall über einige Dateien im Konfigurationsverzeichnis `/etc/shorewall`:

- `/etc/shorewall/zones`: Definiert die Firewall-Zonen, die später in den Regelsätzen verwendet werden können.
- `/etc/shorewall/interfaces`: Regelt, welche Netzwerk-Interfaces mit Shorewall benutzt werden sollen und welchen Zone sie angehören.
- `/etc/shorewall/policy`: Definiert die Default-Policies zwischen Zonen. Darf Traffic zwischen Zonen passieren oder nicht?
- `/etc/shorewall/rules`: Hier werden die feingranularen Regeln definiert: Wurde vorher beispielsweise eine `REJECT` Policy zwischen Zonen definiert, können hier port-/protokoll-basierte Ausnahmen definiert werden. 


Im Folgenden werfen wir einen Blick in unsere Konfiguration:

`zones`:

```
fw              firewall
wan             ipv4
customer1-vpn   ipv4
customer1-vpn   ipv4
mastervpn       ipv4

```

Wir legen Zone `fw` (Alias: `$FW`) als Firewall-eigene Zone fest und definieren weitere Zonen für unsere WAN Schnittstelle und unsere VPN-Netze.


`interfaces`:

```
wan             eth0            detect                  dhcp,routefilter,tcpflags

customer1-vpn   customer1-vpn   detect                  routeback
customer2-vpn   customer2-vpn   detect                  routeback
mastervpn       mastervpn       detect                  routeback
```


Hier werden Netzwerkinterface und Zone miteinander verknüpft - die Zone in der ersten Spalte, das zugeordnete Netzwerkinterface in der zweiten Spalte. Besonders wichtig ist bei den VPN-Zonen die `routeback` Option, die dafür sorgt, dass Traffic, der an einem VPN-Interface eingeht, selbiges auch direkt wieder verlassen kann; so wie es bei einem VPN-Interface die Regel ist, wenn Clients miteinander kommunizieren. Später wird diese Client-to-Client Kommunikation zwar wieder deaktiviert, aber wir wollen uns die Möglichkeit vorhalten, sie für einzelne VPN-Clients zu ermöglichen. 

`policy`:

```
# Source        # Dest  # Policy
wan             all     REJECT
$FW             all     ACCEPT

mastervpn       all     ACCEPT
customer1-vpn   all     REJECT
customer2-vpn   all     REJECT

```


Eingehender Traffic auf WAN soll grundsätzlich erst einmal blockiert werden. Einzelne Ausnahmen - beispielsweise für die VPN-Server-Ports - werden später in der `rules` Datei gemacht. 

Die Firewall selbst bekommt keine Einschränkungen und darf andere Zonen bzw. Interfaces beliebig ansprechen. 

Unser `mastervpn` darf ebenfalls uneingeschränkt kommunizieren. Dabei handelt es sich um das Admin-Interface, von dem aus unsere Entwickler Verbindungen zu verschiedenen VPN-Clients herstellen. 

Restriktiver gehen wir bei den beiden Kunden-VPNs `customer1-vpn` und `customer2-vpn` vor: Diese werden mittels `REJECT` eingesperrt und dürfen per default erst einmal mit niemandem kommunizieren. 


`rules`: Nun folgen die Ausnahmen für die gerade erklärten Policies:

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


Zuerst wird der Zugriff auf die WAN Ports `51821,51822,51823` erlaubt, sodass sich Wireguard-VPN Clients zum VPN Server verbinden können. Außerdem verbinden wir uns über SSH zum VPN-Server - auch das soll weiterhin erlaubt bleiben. In diesem Fall nutzen wir das `SSH(ACCEPT)` Makro von Shorewall, um Port und Protokoll nicht gesondert definieren zu müssen. 

Pings von VPN-Clients zu deren jeweiligen serverseitigen Interface und seiner IP-Adressen sollen in allen Fällen erlaubt sein. Wir pingen in der Regel das Serverinterface an, um die Funktionsfähigkeit einer VPN-Verbindung festzustellen. 

Im nächsten Abschnitt wird Client-to-Client Kommunikation für einzelne VPN-Clients des Netzes `customer1-vpn` explizit ermöglicht. 

... für alle anderen Geräte wird diese Art der Kommunikation im darauf folgenden Abschnitt mittels `REJECT` verhindert. Sie können folglich nur Antwortpakete zurückschicken (implizite Regel in Shorewall) und nicht von sich aus Verbindungen initiieren. 

IP-Forwarding ist auf unserem Server bereits aktiviert. Wer die Einstellung `net.ipv4.ip_forward` in /etc/sysctl.conf noch nicht aktiviert hat, kann stattdessen auch in `/etc/shorewall/shorewall.conf` `IP_FORWARDING` auf `On` setzen. 


Zum Schluss haben wir `netfilter-persistent` aus dem Boot genommen:

	systemctl disable netfilter-persistent

... und die iptables-Regeln geleert: 

	iptables -F

... um dann die neuen Shorewall-Regeln testen zu können:

	shorewall start

Nachdem alles wie erhofft funktioniert hat, wurde Shorewall in den Autostart aufgenommen:

	systemctl enable shorewall
