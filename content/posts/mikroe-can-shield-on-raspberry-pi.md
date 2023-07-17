---
title: "Mikroe CAN SPI Click 3v3 an einem Raspberry Pi 4 betreiben"
date: 2023-06-20T14:31:26+02:00
draft: false
author: "Thomas Leister"
tags: ["can", "raspberrypi", "mikroe", "click"]
---

Die Firma Mikroelektronika (MikroE) aus Serbien bietet innerhalb ihres "Click" Ökosystems zahlreiche Funktionsmodule an, die an Mikrocontrollern betrieben werden können. Eines davon ist das "[CAN SPI Click 3.3V](https://www.mikroe.com/can-spi-33v-click)" CAN Controller Modul, das über die standardisierte "MikroBus" Schnittstelle bzw. SPI verbunden werden kann. Damit die Verbindung zu einem Raspberry Pi 4 funktioniert, haben wir das "[Pi 4 Click Shield](https://www.mikroe.com/pi-4-click-shield)" dazugenommen. 

In diesem Beitrag wollen wir kurz erklären, wie wir das CAN-Modul im Zusammenspiel mit dem Raspberry Pi 4 in Betrieb genommen haben. 

<!--more-->


Das CAN SPI Click 3.3V Modul enthält laut [Produktbeschreibung](https://www.mikroe.com/can-spi-33v-click) einen  MCP2515 CAN controller von Microchip. Damit der Chip von seinem Treiber erkannt werden kann, muss der Device Tree mittels Overlay angepasst werden. Ein passendes Overlay befindet sich bereits auf der Raspi Standdistribution "Raspberry Pi OS", sodass das passende Overlay nur aktiviert werden muss. 

Hierzu wird die Konfigurationsdatei `/boot/config.txt` angepasst:

	[all]
	dtoverlay=mcp2515-can0,oscillator=10000000,interrupt=6

Wichtig sind die beiden Parameter "oscillator" und "interrupt":

* `oscillator` beschreibt die Taktfrequenz des Taktgebers auf dem CAN-Modul. [Der Schaltplan](https://download.mikroe.com/documents/add-on-boards/click/canspi-33v/can-spi-click-33v-manual-v100.pdf) des Moduls offenbart, dass es sich um ein 10 MHz Quarz handelt - also müssen hier 10.000.000 Hz angegeben werden. 
* `interrupt` beschreibt den GPIO-Pin des Raspberry Pi, welcher bei eingehenden CAN-Frames ein Interrupt-Signal übermittelt. Das INT Signal aus eben erwähntem Schaltplan lässt sich über die MikroBus Schnittstelle weiter zum Pi 4 Click Shield verfolgen, wo es schließlich [mit GPIO6 des Raspberry Pi verbunden](https://download.mikroe.com/documents/add-on-boards/click/pi_4_click_shield/pi-4-click-shield-click-schematic-v101.pdf) ist. Daher ist hier der Wert "6" anzugeben.

Weiter geht es mit der Konfiguration der CAN-Netzwerkschnittstelle. In der neuen Datei `/etc/network/interfaces.d/can0` wird folgender Inhalt eingetragen:

	auto can0
		iface can0 can static
  		bitrate 500000

Hierdurch wird auf dem CAN Interface eine Bitrate von 500 kBit/s gesetzt und das Interface beim Boot gestartet. 

Nach einem Reboot des Raspberry Pi sollte in den Kernelmeldungen folgendes zu sehen sein:

	pi@raspi:~ $ dmesg | grep mcp
	[    8.239956] mcp251x spi0.0 can0: MCP2515 successfully initialized.

Der CAN-Controller wurde also erfolgreich erkannt.

Auch ein `ip link` sollte nun ein CAN Interface mit dem Status "UP" zeigen:

	pi@raspi:~ $ ip link
	1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
	    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
	2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
	    link/ether e4:5f:01:af:96:bc brd ff:ff:ff:ff:ff:ff
	3: wlan0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DORMANT group default qlen 1000
	    link/ether e4:5f:01:af:96:bd brd ff:ff:ff:ff:ff:ff
	4: can0: <NOARP,UP,LOWER_UP,ECHO> mtu 16 qdisc pfifo_fast state UP mode DEFAULT group default qlen 10
	    link/can

Um die Funktion zu testen, können wir uns die `can-utils` zunutze machen und dem Raspi selbst eine Nachricht über CAN schicken:

	sudo apt install can-utils

In einer SSH-Session wird das `candump` Kommando gestartet, um alle eingehenden CAN Nachrichten zu zeigen:

	sudo candump can0

In einer weiteren SSH-Session werden Testdaten über `cansend` geschickt:

	sudo cansend 123#FEFE

Wenn die eben verschickte Nachricht auch im candump-Fenster zu sehen ist, funktioniert der CAN-Bus ordnungsgemäß.

---

Quellen: 

* https://download.mikroe.com/documents/add-on-boards/click/pi_4_click_shield/pi-4-click-shield-click-schematic-v101.pdf
* https://download.mikroe.com/documents/add-on-boards/click/canspi-33v/can-spi-click-33v-manual-v100.pdf
* https://download.mikroe.com/documents/datasheets/MCP2515.pdf
* https://crycode.de/can-bus-am-raspberry-pi
* https://www.beyondlogic.org/adding-can-controller-area-network-to-the-raspberry-pi/
