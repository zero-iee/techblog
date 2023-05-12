---
title: "Mobilfunk-Internettraffic außerhalb eines VPNs einschränken"
date: 2023-05-12T15:17:29+02:00
draft: false
author: "Thomas Leister <thomas.leister@zero-iee.com>"
tags: ["iptables", "networking", "wireguard", "lte", "mobilfunk"]
---


In diesem Beitrag wird erklärt, wie wir die `iptables` Firewall eines IoT Gerätes konfiguriert haben, sodass eine Wartung über ein Wireguard VPN-Netz möglich ist, während andere Internetzugriffe aus oder in das Mobilfunknetz verhindert werden. 

<!--more-->

## Gegebenheiten

* (Optional) Ethernetverbindung ins Internet via Lokales LAN (`eth0`)
* (Optional) LTE-Verbindung ins Internet via USB LTE Modem (`usb0`, IP-Adresse `192.168.8.1` bzw. `fe80::c0b0:4fff:fefe:fefe`)
* Wireguard VPN-Server (`vpn.mydomain.tld`)

## Ziel

Die LTE-Verbindung soll ausschließlich für Verbindungen zum Wireguard-VPN-Server genutzt werden können. Nicht als allgemeine Internetverbindung, da der Traffic der SIM-Karte möglichst gering gehalten werden muss (Beschränkung des Datenvolumens!).

Für die Übertragung größerer Datenmengen aus dem Internet (OS-Updates, ... ) soll eine optionale Ethernetverbindung genutzt werden. 

Daher: Beschränkung der ausgehenden Verbindungen über `usb0` Interface auf Verbindungen zum VPN-Server. Weitere Ausnahmen:

* DNS-Anfragen (um Wireguard-Host aufzulösen)
* Ping Requests ins Internet (zum Debugging)

Die LTE-Schnittstelle wird vom Treiber des Netzwerk-Modems automatisch weniger hoch priorisiert als die Ethernet-Schnittstelle. Ist die Ethernet-Schnittstelle verfügbar, werden Daten bevorzugt hierüber (uneingeschränkt) ausgetauscht. 


## Implementierung

`iptables` Firewallregel, um allen ausgehenden Traffic über das `usb0` interface zu blockieren:

	sudo iptables -t filter -A OUTPUT -o usb0 -j REJECT
	sudo ip6tables -t filter -A OUTPUT -o usb0 -j REJECT

_**Wichtig**: Die Blockierregel wird mit `-A` ("append") hinzugefügt. Alle Ausnahmen zu dieser Blockierregel werden im Folgenden mit `-I` ("insert") hinzugefügt, damit sie in der Abarbeitung / Priorisierung weiter vorne stehen und triggern, bevor es zum "REJECT" kommt._

Verbindungen zum LTE Router selbst erlauben - sonst funktioniert gar nichts mehr:

	sudo iptables -t filter -I OUTPUT -o usb0 -d 192.168.8.1 -j ACCEPT
	sudo ip6tables -t filter -I OUTPUT -o usb0 -d fe80::c0b0:4fff:fefe:fefe -j ACCEPT

Allerdings müssen Verbindungen zum VPN-Server erlaubt bleiben:

	sudo iptables -t filter -I OUTPUT -o usb0 -d vpn.mydomain.tld -p udp --dport 51821 -j ACCEPT

* _51821 = Wireguard port_
* _Falls der VPN-Server auch via IPv6 erreichbar ist, Kommando mit `ip6tables` wiederholen_

... und auch zum DNS, denn sonst kann der Hostnamen des Wireguard VPN Servers nicht aufgelöst werden:

	sudo iptables -t filter -I OUTPUT -o usb0 -p udp --dport 53 -j ACCEPT
	sudo ip6tables -t filter -I OUTPUT -o usb0 -p udp --dport 53 -j ACCEPT


## Ausgehende Pings erlauben

Wer zu Debugging-Zwecken ausgehende Pings (ICMP requests) ins LTE-Netz erlauben will, kann eine Ausnahme hinzufügen:

	sudo iptables -t filter -I OUTPUT -o usb0 -p icmp --icmp-type echo-request -j ACCEPT
	sudo ip6tables -t filter -I OUTPUT -o usb0 -p icmpv6 --icmpv6-type echo-request -j ACCEPT


## Alle via Mobilfunk eingehenden Verbindungen blockieren

Um zu verhindern, dass der SSH Port oder ein Webserver direkt aus dem Mobilfunknetz erreicht werden können, wird der Zugriff aus dem Mobilfunknetz weiter eingeschränkt. Services auf dem Target dürfen von außen *nur* über das Wireguard VPN erreichbar sein.

	sudo iptables -t filter -A INPUT -i usb0 -j REJECT
	sudo iptables -t filter -I INPUT -i usb0 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
	sudo ip6tables -t filter -A INPUT -i usb0 -j REJECT
	sudo ip6tables -t filter -I INPUT -i usb0 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

Die `-m conntrack` Regeln bewirken, dass für eingehende Pakete eine Ausnahme gemacht wird, wenn sie zuvor vom Target selbst angefragt wurden. Schließlich sollen Antworten auf DNS-Anfragen oder Ping-Anfragen das Target weiterhin durchdringen können.


## Ergebnis


	pi@target:~ $ sudo iptables -L -t filter -n -v
	Chain INPUT (policy ACCEPT 640 packets, 56736 bytes)
	 pkts bytes target     prot opt in     out     source               destination         
	  456 69620 ACCEPT     all  --  usb0   *       0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
	    0     0 REJECT     all  --  usb0   *       0.0.0.0/0            0.0.0.0/0            reject-with icmp-port-unreachable
	
	Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
	 pkts bytes target     prot opt in     out     source               destination         
	
	Chain OUTPUT (policy ACCEPT 583 packets, 72865 bytes)
	 pkts bytes target     prot opt in     out     source               destination         
	    3   252 ACCEPT     icmp --  *      usb0    0.0.0.0/0            0.0.0.0/0            icmptype 8
	   29  1921 ACCEPT     udp  --  *      usb0    0.0.0.0/0            0.0.0.0/0            udp dpt:53
	  366 77068 ACCEPT     udp  --  *      usb0    0.0.0.0/0            <vpnip>              udp dpt:51821
	    0     0 ACCEPT     all  --  *      usb0    0.0.0.0/0            192.168.8.1         
	   69  6061 REJECT     all  --  *      usb0    0.0.0.0/0            0.0.0.0/0            reject-with icmp-port-unreachable

_`<vpnip>` entspricht hier der aufgelösten IP Adresse zu `vpn.mydomain.tld`. `51821` ist der genutzte Wireguard Port._

Nur Verbindungen, die direkt über das `usb0` Interface gehen, werden von der Firewall beschränkt. Verbindungen, die innerhalb des VPN-Netzes hergestellt werden, bleiben unberührt. So kann beispielsweise unbeschränkt über das Wireguard VPN auf den SSH Port und alle weiteren Dienste auf den Targets zugegriffen werden. 

Falls auch Zugriffe innerhalb des VPNs beschränkt werden sollen, können entsprechende `iptables` Regeln mit einem `-i <wireguardinterface>` Parameter versehen werden, sodass diese nur auf das Wireguard-Interface abzielen. 