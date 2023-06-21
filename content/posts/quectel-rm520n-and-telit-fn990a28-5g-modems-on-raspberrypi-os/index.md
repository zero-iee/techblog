---
title: "Quectel RM520M und Telit FM990A28 5G Modem mit Raspberry Pi OS nutzen"
date: 2023-05-31T12:44:58+02:00
draft: false
author: "Thomas Leister"
tags: ["5g", "connectivity", "raspberrypi", "quectel", "mobilfunk"]
---

Auf unserer Odyssee auf der Suche nach einem 5G Mobilfunkmodem haben wir mittlerweile einige Modems verschiedener Hersteller ausprobiert. Leider war die Inbetriebnahme nicht immer erfolgreich. Mal fehlte der Treibersupport im Linux-Kernel gänzlich - mal war die Ansteuerung via NetworkManager / ModemManager fehlerbehaftet oder überhaupt nicht möglich. 

Uns ist eine einfache Inbetriebnahme und ein stabiler Betrieb wichtig. Da wir die Modems nicht nur auf einigen wenigen Geräten einsetzen wollen, kommt für uns eine manuelle Anpassung des Linux-Kernels in der Regel nicht infrage. Zu groß ist der damit verbundene Aufwand und zu unübersichtlich sind die Folgen, die sich daraus für den weiteren Lebenszyklus eines Produkts ergeben. Daher soll das Betriebssystem - oftmals ein Raspberry Pi OS - möglichst im Werkszustand und ohne große Anpassungen genutzt werden. 

**Mittlerweile haben sich für uns zwei Modems herauskristallisiert, die in Kombination mit dem aktuellen [Raspberry Pi OS](https://www.raspberrypi.com/software/operating-systems/) auf Debian 11 "Bullseye" Basis (Kernel 6.1) "out of the box" funktionieren:**

<!--more--> 

* **[Quectel RM520N](https://www.quectel.com/product/5g-rm520n-gl)**
* **[Telit FM990A28](https://www.telit.com/devices/fn990axx/)**

beide 5G Modems unterstützen nicht nur das aktuell weit verbreitete "5G New Radio" mit LTE Control Plane (NSA), sondern auch 5G NR Standalone (SA), sodass bei einer entsprechenden Ausbaustufe des 5G Mobilfunknetzes auch von extrem niedrigeren Latenzen profitiert werden kann. 


## Hardware Setup


Unser Hardware-Setup für die Evaluierung beider Module sieht wie folgt aus:

* Raspberry Pi CM4 
* [Waveshare Dual Ethernet IoT Base Board](https://www.waveshare.com/wiki/CM4-DUAL-ETH-4G/5G-BASE) (mit M.2 Slot)
* Telit FM990A28 M.2 Modul _oder_
* Quectel RM520N
* IoT SIM-Karte der Deutschen Telekom


{{< figure src="images/waveshare-board.jpg" title="Waveshare Board mit Quectel 5G Modem" >}}


## Inbetriebnahme 

Die Inbetriebnahme unseres Telit-Moduls lief wie folgt ab: _(ähnlich für Quectel!)_

Überprüfen der Sichtbarkeit des Moduls im USB Subsystem:

	$ lsusb

Hier sollte ein Telit Device sichtbar sein: 

	[...]
	Bus 002 Device 003: ID 1bc7:1070 Telit Wireless Solutions FN990
	[...]

Auch der ModemManager sollte das Modul erkennen:

	tom@raspberry:~ $ mmcli -L
    /org/freedesktop/ModemManager1/Modem/0 [Telit] FN990A28

Mittels

	mmcli -m 0

Können einige Details zum Modem ausgegeben werden - unter anderem, ob eine Verbindung zum Mobilfunknetz besteht, oder eine SIM-Karte zugeordnet ist. 

Bei unserem ersten Versuch wurde im "Status" Abschnitt ein rotes `sim-missing` angezeigt, obwohl eine SIM-Karte in den Slot des Waveshare Base IO Moduls eingelegt war. Auch Versuche mit anderen SIM-Karten nutzten nichts - im System wurde keine Karte erkannt. 

### "SIM Missing" Problem beheben

_(dieses Problem betrifft nur das Telit Modem!)_

Ein Blick in die [Schematics des Waveshare Boards](https://www.waveshare.com/w/upload/4/46/CM4-DUAL-ETH-4G_5G-BASE_SchDoc.pdf) offenbarte, dass die Signalleitung ("CD" - "Card detect") für die physische Erkennung einer SIM-Karte im Slot nicht zum M.2 Slot weitergeführt wird, sodass das Mobilfunkmodul kein entsprechendes Signal erkennen _kann_. Kein Wunder also, dass uns permanent ein "**sim-missing**" angezeigt wurde.

{{< figure src="images/waveshare-schematics.png" title="Screenshot des Waveshare Dual Ethernet IoT Base Boards" >}}

Das Problem lässt sich beheben, indem das Modem so konfiguriert wird, dass es eine immer eingelegte SIM-Karte annimmt und keine HotSwap-Abfragen mehr durchführt. Die Konfiguration geschieht über AT-Kommandos innerhalb einer Terminalsession mit dem Modem selbst:

	sudo apt install minicom
	sudo minicom -D /dev/ttyUSB2

AT-Kommandos absetzen - sollten mit "OK" quittiert werden.

	AT#HSEN=0,0
	AT#HSEN=0,1

_(genauer genommen wird hier HotSwap für beide potentiellen SIM-Steckplätze, die das Modem unterstützt, deaktiviert.)_

Die Minicom Session kann mit CTRL-A gefolgt von "X" beendet werden.

Damit die Änderung angewendet wird, wurde das Raspi samt Modem neu gestartet / die Stromversorgung unterbrochen. 

Nach einem Neustart wurde die SIM-Karte im ModemManager schließlich erkannt - am Ende der Ausgabe von `mmcli -m 0` wurde eine "SIM" Zeile mit dem D-Bus Pfad zum SIM-Device eingeblendet:

	SIM      |            dbus path: /org/freedesktop/ModemManager1/SIM/0


**Die soeben erwähnte Anpassung am Telit Modem ist am Quectel-Modem nicht notwendig!**

Im nächsten Schritt wird das jeweiligen Modem aktiviert:

	mmcli -m 0 --enable


### Eine Verbindung mittels NetworkManager einrichten

Mittels NetworkManager wird nun eine neue Mobilfunkverbindung eingerichtet. Dazu legen wir im NetworkManager eine neue "Connection" an. Im Hintergrund kommuniziert dieser mit dem ModemManager, um APN-Details an ihn weiterzugeben.

Für unsere Telekom-Karte sind folgende APN-Informationen zu nutzen:

* APN: internet.telekom
* IP-Type: ipv4
* Username: telekom
* Password: tm

Die APN-Informationen eines jeden Providers lassen sich schnell im Internet nachschlagen.

Mit dem neueren IPv6-fähigen APN der Telekom `internet.v6.telekom` (und "ipv4v6") hatten wir leider kein Glück - wir konnten keine Vebindung herstellen. Probleme im IPv6-Stack der Modemtreiber sind uns bereits bekannt. Evtl. kommen sie auch hier zum tragen. Daher begnügen wir uns vorerst mit reinem IPv4 Support.

Bevor die Verbindung angelegt werden kann, muss sichergestellt sein, dass der NetworkManager läuft: 

	systemctl enable --now NetworkManager

Auf einem Stock Raspberry Pi OS Image ist dies nicht der Fall. Ein Reboot nach dem systemctl Kommando kann nicht schaden. In unserem Fall funktionierte das Zusammenspiel zwischen den beiden Managern erst nach einem Reboot. 

Schließlich wird mit diesem Kommando eine neue GSM Verbindung im NetworkManager angelegt:

	mmcli c add type gsm ifname cdc-wdm0 con-name telekom apn internet.telekom connection.autoconnect yes

Im Falle der Telekom ist der APN "internet.telekom" inkl. der übrigen Parameter bereits im SIM-Profil hinterlegt, sodass nur noch der Name des passenden APN-Profils angegeben werden muss. Auf Benutzername und Passwort kann idR verzichtet werden. 

Sollte das nicht funktionieren, können alternativ auch weitere Parameter mitgegeben werden, wie z.B. 

	mmcli c add type gsm ifname cdc-wdm0 con-name telekom apn internet.telekom gsm.username telekom gsm.password tm gsm.pin 1234 connection.autoconnect yes

Insbes. die Angabe einer `gsm.pin` ist wichtig, falls die SIM-Karte mit einer PIN geschützt ist. Unsere SIM ist nicht mit einer PIN geschützt, daher entfällt die Angabe.

Ein `nmcli c` sollte nun zeigen, dass eine neue Verbindung "telekom" angelegt wurde. Ist die Verbindung grün markiert, hat die Anmeldung im Netzwerk funktioniert:

{{< figure src="images/networkmanager-ok.png" title="nmcli c" >}}

Auch der ModemManager sollte nach einer kurzen Weile ähnlich wie hier aussehen und ein "connected" im Status zeigen: _(`mmcli -m 0`)_

{{< figure src="images/modemmanager-ok.png" title="mmcli -m 0" >}}

### Funktion prüfen

Ob die Mobilfunkverbindung tatsächlich funktioniert, lässt sich schnell und einfach über einen Ping auf dem Mobilfunk-Device prüfen:

	ping -I wwan0 1.1.1.1

Die Latenz bewegt sich bei einem 5G NSA Netz üblicherweise bei >= 25 ms, kann aber stark schwanken. Wir haben Latenzen von bis zu 600 ms beobachtet - abhängig von Empfang und Auslastung des Netzwerks. 

Bei einem Neustart des Raspberry Pis wird die Mobilfunkverbindung automatisch neu aufgenommen. 

Übrigens: Ein `ip route` offenbart, dass NetworkManager eine Defaultroute für das 5G Modem angelegt hat. Da diese aber mit einer Metrik von 700 versehen ist, wird auf das Mobilfunkmodem  nur zurückgegegriffen, falls ein Ziel über eine möglicherwiese vorhandene Ethernetverbindung nicht erreichbar ist. Ein `apt update` und alles andere sollte also normalerweise über eine verfügbare Ethernetverbindung laufen. Ist diese nicht verfügbar, dient die Mobilfunkverbindung als Fallback (daher der `-I wwan0` Parameter im `ping` Kommando - hiermit wird eine Verbindung via Mobilfunk erzwungen).

 
