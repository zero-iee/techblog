---
title: "Standard DNS Resolver für Dnsmasq auf Raspbian Buster festlegen"
date: 2023-06-12T12:43:53+02:00
draft: false
author: "Thomas Leister"
tags: ["dns", "dnsmasq", "raspbian", "resolvconf"]
---

Auf einem unserer Raspberry Pis mit Raspbian "Buster" Image hatten wir in Kombination mit einem USB-Mobilfunkstick ein merkwürdiges Problem: Der Wireguard VPN Client konnte sich immer wieder beim Starten des Pis nicht korrekt mit dem Wireguard Server verbinden. Das Fehler-Log besagte, dass der Hostname des Wireguard Servers in der Clientkonfiguration nicht aufgelöst werden konnte. 

Eine mögliche Ursache dafür könnte sein, dass zum Startzeitpunkt des Wireguard-Servers der interne DNS-Resolver des Mobilfunksticks (NAT-/Routerbetrieb) noch nicht einsatzbereit war und die Auflösung deswegen scheiterte. Um die Theorie zu bestätigen und den Fehler zu beheben, sollte nun ein Default-Nameserver eingeführt werden, der unabhängig von der Netzwerkverbindung immer derselbe ist und sofort zur Verfügung steht.

<!--more-->

Üblicherweise wird der derzeit genutzte DNS-Resolver vom Netzwerkmanagement automatisch in die Datei `/etc/resolv.conf` eingetragen - so auch bei Raspbian. In den meisten Fällen wird hier der DNS-Resolver eingetragen, der vom DHCP-Server des lokalen Netzwerks zugewiesen wurde. 

In unserem Fall gestaltet sich das Setup etwas komplizierter: 
Da wir zu anderen Zwecken Dnsmasq auf dem Raspi betreiben, hat dieses die Kontrolle  über die DNS-Namensauflösung übernommen und sich selbst (127.0.0.1) in die `/etc/resolv.conf` eingetragen. Dnsmasq selbst übernimmt allerdings nur die Rolle eines "caching DNS resolvers" - es stellt also nicht selbst Anfragen bis zu den DNS Rootservern, sondern nutzt seiterseits einen weiteren, externen DNS-Resolver. 

Doch welcher DNS-Resolver wird von Dnsmasq angesprochen? 
Die Antwort findet sich nicht - wie zunächst erwartet - in der Dnsmasq Konfiguration unter `/etc/dnsmasq`. Ein Blick in Tabelle der laufenden Prozesse offenbart, dass Dnsmasq mit der Option `-r` gestartet wurde:

	$ sudo ps -aux
	
	dnsmasq    647  0.0  0.1  11076  1876 ?        S    10:12   0:00 /usr/sbin/dnsmasq -x /run/dnsmasq/dnsmasq.pid -u dnsmasq -r /run/dnsmasq/resolv.conf -7 /etc/dnsmasq.d,.dpkg-dist,

`-r` steht für `--resolv-file` und zeigt auf eine Datei `/run/dnsmasq/resolv.conf`, die die Upstream DNS Resolver enthält, auf welche Dnsmasq zurückgreifen soll. 

Ein Blick in die Datei verrät, dass der DNS-Resolver unseres ISP dort eingetragen wurde. Die erste Zeile weist darauf hin, dass die Datei von `resolvconf` generiert wirde. Doch wie können wir unseren eigenen Standard-Resolver hier hinterlegen?

In StackOverflow Antworten wie [dieser](https://unix.stackexchange.com/questions/128220/how-do-i-set-my-dns-when-resolv-conf-is-being-overwritten) wird empfohlen, den Default-Nameserver in die Datei `/etc/resolvconf/resolv.conf.d/head` oder `base` einzutragen. Auf unserem Target scheitert das jedoch - schon das Verzeichnis `/etc/resolvconf/resolv.conf.d` ist nicht zu finden. 

Der Grund liegt darin, dass Raspbian "Buster" eine anderen `resolvconf` Implementierung nutzt, als neuere Linux-Distributionen: `openresolv`. Diese kennt keine `head` oder `base` Datei. Dennoch ist es möglich, einen oder mehrere Default-Resolver zu hinterlegen, welche automatisch in die Liste der zu nutzenden Resolver aufgenommen werden.

Hierzu muss die Konfigurationsdatei `/etc/resolvconf.conf` bearbeitet und eine Zeile wie z.B. die Folgende hinzugefügt werden:

	name_servers="9.9.9.9"

Alternativ für mehrere Server z.B. 

	name_servers="9.9.9.9 1.1.1.1 8.8.8.8"

Um die Änderungen anzuwenden, wird Dnsmasq's Resolverdatei `/run/dnsmasq/resolv.conf` neu generiert:

	sudo resolvconf -u /run/dnsmasq/resolv.conf

Ein Check der Datei zeigt, dass die Default-Resolver (neben dem netzwerkspezifischen Resolver `10.0.0.1`) aufgenommen wurde: 

	nameserver 9.9.9.9
	nameserver 10.0.0.1

In diesem Beispiel wurde der Quad9-Server 9.9.9.9 aufgenommen. Um zu prüfen, ob dieser nun tatsächlich bei einer Namensauflösung angesprochen wird, kann folgendes Kommando abgesetzt werden:

_(zuvor evtl. `apt install dnsutils`)_

	nslookup -q=txt -class=chaos id.server.on.quad9.net

Die Antwort sollte in etwa wie folgt aussehen:

	;; Warning: Message parser reports malformed message packet.
	Server:		127.0.0.1
	Address:	127.0.0.1#53
	
	Non-authoritative answer:
	id.server.on.quad9.net	canonical name = res120.fra.on.quad9.net.
	
	Authoritative answers can be found from:


Bedeutend ist hierbei, dass der "canonical name" auf "quad9.net" endet. Wird stattdessen eine "SERVFAIL" Antwort zurückgegeben, ist etwas schiefgelaufen und der DNS-Resolver offenbar nicht aktiv.

