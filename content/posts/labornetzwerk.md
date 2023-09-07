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

Das Labornetz am Netzwerkinterface `ens37` bekommt das IPv4-Netz `10.0.0.1/24`. IP-Adressen werden vergeben zwischen 10.0.0.10 und 10.0.0.254. Die eigene Entwicklungsrechner soll darin als Router und DNS-Resolver agieren und besitzt die IP-Adresse `10.0.0.1`.

Die Netzwerkparameter auf dem Laborinterface werden in den Netzwerkeinstellungen statisch festgelegt:

* IP-Adresse: 10.0.0.1
* Netzmaske: /24 (255.255.255.0)
* Gateway: 10.0.0.1

## Dnsmasq als DHCP- und DNS-Server einsetzen

Der `dnsmasq` Server wird genutzt, um einen DHCP- und DNS-Server im Labornetz bereitzustellen. DNS wird benötigt, falls über NAT (siehe letzter Abschnitt) eine Internetverbindung genutzt werden soll.

	sudo apt install dnsmasq

In der neuen Konfigurationsdatei`/etc/dnsmasq.d/labnet.conf` wird dnsmasq konfiguriert:

	# Interface to use
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

`interface=ens37` sorgt dafür, dass der DNS-Resolver nur auf dem Netzwerkinterface `ens37` horcht. Lässt man die Einstellung weg, versucht `dnsmasq`, auch auf dem localhost Interface einen DNS-Service zu starten, was auf modernen Linuxdistributionen fehlschlägt, da hier schon `systemd-resolved` läuft. 

Darauf folgend wird der DHCP-Bereich definiert, aus dem IP-Adressen an Geräte im Labornetz verteilt werden sollen (`10.0.0.100 - 10.0.0.250`). Dabei wird garantiert, dass eine IP-Adresse mindestens 12 Stunden lang ihre Gültigkeit behält.

Über die `dhcp-option` Einstellungen wird den abgeschlossenen Geräten mitgeteilt, unter welchen IP-Adressen sich das Standarddateway und der DNS-Resolver befinden. In beiden Fällen handelt es sich um den Rechner, der dnsmasq ausführt.

Mit `dhcp-host` können IP-Adressen für einzelne Netzwerkinterfaces bzw. Geräte "fixiert" werden. So wird dem Gerät mit der MAC-Adresse `10:00:00:00:00:0` immer die IP-Adresse `10.0.0.101` zugeordnet.

Sollen die im Labornetz angeschlossenen Geräte Internetzugriff bekommen (siehe Abschnitt _"Eine (temporäre) Internetverbindung bereitstellen"_), muss für die dnsmasq-Instanz ein Upstream-DNS Server festgelegt werden. Dnsmasq kann globale Domainnamen wie z.B. google.com nicht selbst auflösen und greift dafür auf den hier hinterlegten DNS-Server zurück. Hier wird er auf die IP-Adresse des systemd-resolved Services gesetzt, welcher mit den meisten modernen Linuxdistributionen mitgeliefert wird.

Die drei folgenden Parameter (`domain=` ff.) legen fest, welche Top-Level-Domain innerhalb des Labornetzes genutzt werden soll. Das ist wichtig, um systemd-resolved später mitteilen zu können, welcher DNS-Resolver (näcmlich dnsmasq!) für das Labornetz genutzt werden soll. 

Wer öffentliche DNS-Einträge manipulieren oder eigene, virtuelle simulieren will, kann eine oder mehrere `address=` Parameter setzen. Der darin erwähnte Domainnamen wird dann statisch in die dahinter gesetzte IP-Adresse umgesetzt (und der Upstream-DNS-Server umgangen). 


Die neue Config-Datei wird unten in der Datei `/etc/dnsmasq.conf` noch aktiviert, indem folgende Zeile einkommentiert wird:

	conf-dir=/etc/dnsmasq.d/,*.conf

Zu guter Letzt wird systemd-resolved mitgeteilt, dass es Domainnamen mit der Endung `.lab` automatisch mithilfe des dnsmasq-Servers auflösen soll, statt mit einem anderen ggf öffentlichen DNS-Server (welcher die Namen im Labornetz selbstverständlich nicht kennen würde):

	sudo resolvectl domain ens37 lab
	sudo resolvectl dns ens37 10.0.0.1


Nach einem Neustart von `dnsmasq` ziehen sich Geräte, die nur am Laborinterface angesteckt werden, eine IP-Adresse via DHCP und sollten von der Entwicklungsrechner aus erreichbar sein. Welche IP-Adresse ein Gerät bekommen hat, lässt sich im `dnsmasq` Log nachvollziehen:

	sudo journalctl -u dnsmasq -f

Meldet sich ein Gerät mit dem Hostnamen `mydevice` am DHCP-Server, kann es über den Hostnamen `mydevice.lab` beispielsweise angepingt werden oder eine SSH-Sitzung wie folgt eröffnet werden:

	ssh user@mydevice.lab


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
