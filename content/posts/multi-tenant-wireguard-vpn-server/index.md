---
title: "Einen Multi-Tenant Wireguard VPN Server mit iptables absichern"
date: 2023-11-29T11:31:59+01:00
draft: false
author: "Thomas Leister"
tags: ["wireguard", "vpn"]
---

Die [ZERO AMPS Nodes](https://www.zero-iee.com/de/products/) verfügen zwar nicht per default über eine Internetverbindung, doch in einigen Fällen statten wir sie mit einem Mobilfunkmodul aus, sodass wir sie in Absprache mit dem Kunden aus der Ferne aktualisieren, warten oder Fehler beheben können. 

Um eine sichere Verbindung zu unserer eigenen Infrastruktur herzustellen, nutzen wir größtenteils [Wireguard](https://www.wireguard.com) VPNs. Wireguard VPN sind sehr leichtgewichtig, performant und sind sich erfahrungsgemäß sehr robust - vor allem in Kombination mit Mobilfunkverbindungen. Der Wireguard Client auf den AMPS Nodes verbindet sich zu unserem zentralen VPN-Server. Über diesen stellen auch unsere Entwickler eine Verbindung her, sodass sie sich letztendlich zu dem jeweiligen AMPS Node verbinden können. 

<!--more-->

Da wir allerdings verschiedene Kunden mit verschieden großen Projekten haben und bei der Datensicherheit keine Kompromisse eingehen wollen, haben wir unsere VPN-Netze nach Kunden und Projekten aufgetrennt:

Kunde A wird so mit seinen Geräten niemals mit Geräten des Kunden B in Kontakt kommen können - dafür sorgen die individuellen VPN-Netze und unsere Firewallregeln, die den Datenverkehr innerhalb der VPNs und darüber hinaus beschränken. Es muss unbedingt verhindert werden, dass ein kompromittiertes Gerät die Kontrolle über sämtliche - auch kundenfremde - Nodes erhält. 

In der Regel kann also von einem Node aus nicht aus einem Kunden-VPN heraus kommuniziert werden. Eine Ausnahme gibt es dann aber doch: Ein eigenes "Master VPN" (`mastervpn`) erlaubt unseren Entwicklern, über dieses VPN Geräte _aller anderen_ VPNs zu erreichen. Statt sich also mit vielen verschiedenen Wireguard-Profilen herumschlagen zu müssen, genügt es, wenn unsere Entwickler eine Verbindung mit diesem einen Master VPN herstellen. 

Unsere Netzwerktopologie sieht also in etwa so aus: 

![Networks Graphics](images/wireguard-networks.svg)


Die Firewallregeln auf dem VPN-Server vor der Implementierung der Sicherheitsmaßnahmen:

```
root@vpnserver:~# iptables -L -v
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination                 

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination 
```

_(Grund für die gähnende Leere: eine weitere Firewall befindet sich außerhalb des VPN-Servers und filtert eingehende Verbindungen bereits)_


Diese Regeln galt es nun so auszubauen, dass folgende Anforderungen erfüllt sind:

* Ein AMPS Node in einem der Wireguard-Netz darf nur mit anderen AMPS Nodes (oder dem Server-Interface) seines eigenen VPN-Subnetzes kommunizieren. 
* Ein Gerät im `mastervpn` Wiregard Netz darf mit _beliebigen_ AMPS Nodes in _allen_ Wireguard-Netzen kommunizieren. 


Damit überhaupt eine Kommunikation / ein Paket-Forwarding über Netzgrenzen (z.B. `mastervpn` => `customer1-vpn` oder `mastervpn` => `customer2-vpn`) hinweg möglich ist, muss im Linux Kernel zuerst das IPv4-Forwarding aktiviert werden. Temporär via 

	sysctl -w net.ipv4.ip_forward=1

... oder dauerhaft durch die Anpassung der Datei `/etc/sysctl.conf`:

	net.ipv4.ip_forward=1

Gefolgt durch ein 

	sysctl -p

Per default kann nun jedes Netzwerkinterface an jedes andere Datenpakete weiterleiten. Clients verschiedener Wireguard-Netze könnten also - wenn sie clientseitig passend konfiguriert sind -  miteinander sprechen. Doch dieses Verhalten soll bei uns normalerweise _nicht_ erlaubt sein. Daher wird die Default-Firewallregel für die "FORWARD" Chain der "Filter" Tabelle auf "DROP" gesetzt:

	iptables -P FORWARD DROP

Für einen Fall müssen wir das Forwarding allerdings erlauben - nämlich für den Fall, dass eine Anfrage aus dem `mastervpn` heraus in eines der Kunden-VPNs geschickt wird. Für diesen Fall wurde das Forwarding ja zuvor aktiviert:

	iptables -A FORWARD --in-interface mastervpn --out-interface customer1-vpn -j ACCEPT
	iptables -A FORWARD --in-interface mastervpn --out-interface customer2-vpn -j ACCEPT


Damit Geräte aus dem betreffenden Kunden-VPN antworten können, ist eine weitere Firewallregel wichtig, die dafür sorgt, dass AMPS Nodes aus dem Kunden-VPN in einem Fall eben doch Pakete in ein anderes Wireguard-Netz (`mastervpn`) schicken dürfen. Nämlich dann, wenn es sich um eine Antwort auf eine zuvor eingegangene Anfrage aus `mastervpn` handelt:

	iptables -A FORWARD --out-interface mastervpn -m state --state ESTABLISHED,RELATED -j ACCEPT


Bisher haben wir nur die `FORWARD` Regeln betrachtet. Dabei fällt aber ein Fall durch's Raster: Was, wenn ein Paket nicht weitergeleitet werden muss, sondern sein Ziel schon erreicht hat? Das ist beispielsweise der Fall, wenn von `customer3-vpn` aus die _Wireguard Server-IP_ von `customer2-vpn` angesprochen wird. Da hier - trotz anders lautender IP-Adresse der Server selbst angesprochen wird, kommt kein Forwarding zum Tragen und derartige Kommunikation wird von den bisherigen Regeln nicht verhindert.

Wenn wir solche Anfragen verhindern wollen, muss eine `INPUT` Regel für jedes der VPNs erzeugt werden, z.B.:

	iptables -A INPUT -d 10.4.0.1 ! --in-interface mastervpn -j REJECT
	iptables -A INPUT -d 10.2.0.1 ! --in-interface customer1-vpn -j REJECT
	iptables -A INPUT -d 10.3.0.1 ! --in-interface customer2-vpn -j REJECT

`10.4.0.1` ist hier beispielsweise die IP-Adresse, die innerhalb des `mastervpn` für den Server selbst genutzt wird. Genauso verhält es sich bei den anderen beiden Regeln für die `customer` VPNs.

Firewallregeln nach der Implementierung der Sicherheitsmaßnahmen:

```
root@vpnserver:~# iptables -L -v
Chain INPUT (policy ACCEPT 5621 packets, 1148K bytes)
 pkts bytes target     prot opt in     out     source               destination         
    7   588 REJECT     all  --  !mastervpn any     anywhere             10.4.0.1             
    0   0   REJECT     all  --  !customer1-vpn any     anywhere             10.2.0.1            
    0   0   REJECT     all  --  !customer2-vpn any     anywhere             10.3.0.1                
reject-with icmp-port-unreachable

Chain FORWARD (policy DROP 17 packets, 1428 bytes)
 pkts bytes target     prot opt in     out     source               destination                   
    2   168 ACCEPT     all  --  mastervpn customer1-vpn  anywhere             anywhere            
  207 17201 ACCEPT     all  --  mastervpn customer2-vpn  anywhere             anywhere            
   38  4692 ACCEPT     all  --  any    mastervpn  anywhere             anywhere             state RELATED,ESTABLISHED

Chain OUTPUT (policy ACCEPT 6708 packets, 1425K bytes)
 pkts bytes target     prot opt in     out     source               destination
```



Der Weg durch die Firewall wäre hiermit für unsere AMPS Kundengeräte geebnet. 

Nur eines fehlt noch: Eine Routinginformation für den Fall, dass jemand aus dem `mastervpn` heraus auf eines der AMPS Nodes in den `customer` VPNs zugreift. Es muss spezifiziert werden, wohin Antwortpakete geschickt werden sollen. Schließlich sehen die Geräte nur eine IP-Adresse aus dem mastervpn-Bereich, z.B. `10.4.0.0/24`. Da standardmäßig nur Routinginformationen für ihr eigenes Subnetz vorliegen, müssen wir ihnen den Weg zu 10.4.0.0/24 zeigen. Dazu wird in der jeweiligen Wireguard-Konfiguration auf dem AMPS Node ein Eintrag für `AllowedIPs` hinzugefügt:

Aus 

	AllowedIPs = 10.3.0.0/16

wird beispielsweise

	AllowedIPs = 10.3.0.0/16,10.4.0.0/24

Somit ist für den betroffenen Wireguard-Client aus dem Kundennetz klar, dass nicht nur Pakete für das eigene SUbnetz über den Server geroutet werden sollen, sondern auch (Antwort-)Pakete an das `mastervpn` Netz `10.4.0.0/24`.

Nach einem Neuladen der Clientkonfiguration mittels `systemctl restart wg-quick@customer1-vpn` ist die Änderung aktiv und die Routinginformationen sollten vorliegen:

	ip route


_Hinweis: Die oben genannten iptables Einstellungen sind **nicht** persistent! Um sie nach einem Reboot wiederherzustellen, empfehlen wir, iptables Regeln mittels `netfilter-persistent` zu persistieren._