---
title: "Ein Labornetzwerk unter Linux einrichten"
date: 2023-07-21T12:18:27+02:00
draft: false
author: "Thomas Leister"
tags: ["embedded", "netzwerk", "linux"]
---

Während meiner Arbeit verbinde ich mich regelmäßig zu verschiedenen Rechnern und eingebetteten Geräten, die über eine Ethernetverbindung erreichbar sind. Diese könnte man nun direkt - wie den Entwicklungsrechner auch - an das Firmennetzwerk anschließen...

... oder man erstellt für seine Geräte ein eigenes "Labornetz", welches nur vom eigenen Laptop aus erreichbar ist und über das man die volle Kontrolle hat. Vorteile können sein:

* Überblick über die verbundenen Geräte und ihre IP-Adressen
* Keine Exposition der angeschlossenen Geräte ins Firmennetz (Verbesserung der Sicherheit)
* Ist am Entwicklungsrechner nur WLAN verfügbar, können die embedded Geräte trotzdem einfach und kabelbunden erreicht werden.

In größeren Unternehmen kann zudem der Zugang zum internen Firmennetzwerk stark reguliert sein, sodass sich überhaupt nur freigeschaltene Geräte nutzen lassen. Mit einem eigenen kleinen Labornetz auf einem zweiten Netzwerkinterface kann das Problem elegant umgangen werden.

<!--more-->


## Der Plan

Das Labornetz bekommt das IPv4-Netz "10.0.0.1/24". IP-Adressen werden vergeben zwischen 10.0.0.10 und 10.0.0.254. Die eigene Entwicklungsrechner soll darin als Router und DNS-Resolver agieren und besitzt die IP-Adresse 10.0.0.1.

Die Netzwerkparameter auf dem Laborinterface werden in den Netzwerkeinstellungen statisch festgelegt:

* IP-Adresse: 10.0.0.1
* Netzmaske: /24 (255.255.255.0)
* Gateway: 10.0.0.1

## Dnsmasq als DHCP- und DNS-Server einsetzen

Der `dnsmasq` Server wird genutzt, um einen DHCP- und DNS-Server im Labornetz bereitzustellen. DNS wird benötigt, falls über NAT (siehe letzter Abschnitt) eine Internetverbindung genutzt werden soll.

	sudo apt install dnsmasq

In der neuen Konfigurationsdatei`/etc/dnsmasq.d/labnet.conf` wird `dnsmasq` konfiguriert:

	listen-address=10.0.0.1
	bind-interfaces
	dhcp-range=10.0.0.10,10.0.0.254,255.255.255.0,24h
	dhcp-option=option:router,10.0.0.1
	dhcp-option=option:dns-server,10.0.0.1

`listen-address` und `bind-interfaces` sorgen dafür, dass der mitgelieferte DNS-Resolver nur auf dem Netzwerkinterface horcht, welches die IP-Adresse 10.0.0.1 zugeordnet hat. Lässt man die Option `bind-interfaces` weg, versucht `dnsmasq`, auch auf dem localhost Interface einen DNS-Service zu instaltiieren, was auf modernen Linuxdistributionen fehlschlägt, da hier schon `systemd-resolved` läuft. 

Die neue Config-Datei wird unten in der Datei `/etc/dnsmasq.conf` noch aktiviert, indem folgende Zeile einkommentiert wird:

	conf-dir=/etc/dnsmasq.d/,*.conf

Nach einem Neustart von `dnsmasq` ziehen sich Geräte, die nur am Laborinterface angesteckt werden, eine IP-Adresse via DHCP und sollten von der Entwicklungsrechner aus erreichbar sein. Welche IP-Adresse ein Gerät bekommen hat, lässt sich im `dnsmasq` Log nachvollziehen:

	sudo journalctl -u dnsmasq -f

## Eine (temporäre) Internetverbindung bereitstellen

Damit angeschlossene Geräte für Softwareinstallationen und Updates einen Weg ins Internet finden, kann auf der Entwicklungsrechner ein Source-NAT eingerichtet werden. Ich nutze hierfür ein kleines Script:

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

_(Wichtig: Interfacebezeichnungen `ens33` bzw `ens37` anpassen via Variablen EXT_IFACE und INT_IFACE!)_

Das Script wird unter `~/.local/bin/natctl` abgespeichert und im PATH bekannt gemacht:

`~/.bashrc`:

	export PATH=$PATH:~/.local/bin
	
Außerdem wird das Script ausführbar gemacht:

	chmod u+x  ~/.local/bin/natctl

Nach einem `source ~/.bashrc` sollte das `natctl` Script verfügbar sein. Über zwei einfache Kommandos kann die Internetverbindung dann aktiviert bzw. deaktiviert werden:

	natctl start
	natctl stop


_Hinweis: Nach einem Reboot muss der `dnsmasq` Server evtl. neu gestartet werden, da das Laborinterface zum ersten Startversuch möglicherweise noch nicht die passende IP-Adresse hatte und dieser deshalb fehlgeschlagen sein könnte. Sollten neu angesteckte Geräte also keine IP-Adresse bekommen, lohnt es sich, einmal einen Neustart via `sudo systemctl restart dnsmasq` zu versuchen._

_Hinweis 2: Auch die NAT-Funktion überlebt (absichtlich) keinen Neustart. Nach dem Booten muss das NAT mittels `natctl start` wieder aktiviert werden._
