---
title: "Waveshare CM4 Dual Gigabit Ethernet 5G/4G Base Board mit Radxa Rock 3 CM3 Compute Module in Betrieb nehmen"
date: 2023-03-22T09:56:25+01:00
draft: false
author: "Thomas Leister"
tags: ["rockship", "radxa", "raspi", "raspberrypi", "cm4", "cm3", "waveshare", "computeModule"]
---


Da da Raspberry Pi CM4 ("Compute Module") seit längerer Zeit wegen der anhaltenden Lieferkettenprobleme schwer und nur in geringen Stückzahlen zu beschaffen ist, haben wir von der ZERO GmbH uns nach einer Alternative umgesehen. 

Obwohl der Name es nicht direkt vermuten lässt, bietet Radxa mit dem "[Rock3 CM3](https://wiki.radxa.com/Rock3/CM/CM3)" eine weitestgehend Pin-kompatible Alternative zum Raspberry Pi CM4 an. _"Weitestgehend"_, da Radxa über einen zusätzlichen, dritten Sockel weitere Pins für Radxa-spezifische Features anbietet. Weitere Interschiede sind auf [dieser Seite in der Tabelle](https://wiki.radxa.com/Rock3/CM3/vsCM4) zu finden. Dennoch lässt sich das Radxa CM3 Modul hardwareseitig problemlos in ein bestehendes Raspberry Pi CM4 IO-Board einbauen - in unserem Fall ein [Waveshare Dual Gigabit Ethernet 5G/4G Base Board](https://www.waveshare.com/CM4-DUAL-ETH-4G-5G-BASE.htm). 

Auch wenn die Schnittstelle zwischen CM und Base IO Board nahezu identisch ist - der SoC auf dem Compute Module ist es keinesfalls. Während das CM4 der Raspberry Foundation auf einen Broadcom SoC (BCM2711) setzt, kommt auf dem Radxa Rock3 CM3 ein Rockchip-SoC (RK3566) zum Einsatz. Die Software kann also nicht 1:1 übertragen werden. Ein Raspbian oder Raspberry Pi OS auf dem Rockchip auszuführen, ist also nicht möglich.

Stattdessen bietet Radxa ein paar für den Rockchip geeignete Betriebssysteme zum Download an. Darunter auch eine Debian 11 ("Bullyeye") Version, deren Kernel auf den Rockchip-SoC angepasst ist. Leider kommt die Linux-Distribution mit dem Uraltkernel 4.19 - das soll hier aber erst einmal nicht weiter stören. 

Die [Downloadseite](https://wiki.radxa.com/Rock3/downloads) verweist auf GitHub Releaseseiten. Von dort aus kann die passende Debian-Version heruntergeladen werden. Da wir allerdings nicht das Radxa-eigene IO Board nutzen, sondern das Waveshare IO Board, welches für ein original CM4 konzipiert wurde, ist beim Download Vorsicht geboten: Hier muss die Version für ein "RASPCM4IO" board heruntergeladen werden. Das Image `radxa-cm3-io-debian-bullseye-xfce4-arm64-<VERSION>-gpt.img.xz` ist also das passende (GitHub: [Download](https://github.com/radxa-build/radxa-cm3-io/releases/tag/20221101-0118)).

Nun aber zu den konkreten Schritten für das Aufspielen des Debian-Images auf den eMMC Flash des Rock3 CM3 - unter Ubuntu Linux als Host. ...


## rkdeveloptool herunterladen und installieren

Da wir auf eine SD-Karte verzichten und das OS-Image direkt in den onboard-eMMC Speicher des CM3 schreiben wollen, benötigen wir zunächst das passende Entwicklerwerkzeug von Rockchip, mit dem sich derartige Schreibbvorgänge umsetzen lassen. 

Das passende Tool heißt `rkdeveloptool` und kann von GitHub heruntergeladen werden:

	git clone https://github.com/rockchip-linux/rkdeveloptool.git

Für den Kompiliervorgang müssen einige Pakete auf dem Host installiert sein:

	sudo apt-get install build-essential libudev-dev libusb-1.0-0-dev dh-autoreconf

Danach wird es aus dem Sourcecode kompiliert und installiert: 

	cd rkdeveloptool
	autoreconf -i
	./configure
	make
	sudo make install


## OS-Image in eMMC schreiben

Nun wird auf dem Waveshare IO Board der "Boot" Schalter auf "on" gestellt und das Board mit einem USB-C-Kabel zum Hostrechner verbunden. Ein `lsusb` auf der Kommandozeile sollte offenbaren, dass ein Gerät von "Fuzhou Rockchip Electronics Company" erkannt wurde:

	Bus 003 Device 005: ID 2207:350a Fuzhou Rockchip Electronics Company 

Nun laden wir uns den Rockchip Loader herunter; eine Softwarekomponente, die in den RAM des SoC geladen wird, sodass wir von dort aus das Debian-Image in den eMMC Flash schreiben können:

	wget https://dl.radxa.com/rock3/images/loader/rock-3a/rk356x_spl_loader_ddr1056_v1.10.111.bin

Der Loader wird nun über das rkdeveloptool in den RAM des SoC übertragen:

	sudo rkdeveloptool db rk356x_spl_loader_ddr1056_v1.10.111.bin

Nun kann das Debian OS Image von GitHub heruntergeladen und entpackt werden:

	wget https://github.com/radxa-build/radxa-cm3-io/releases/download/20221101-0118/radxa-cm3-io-debian-bullseye-xfce4-arm64-20221101-0302-gpt.img.xz
	
	xz -d radxa-cm3-io-debian-bullseye-xfce4-arm64-20221101-0302-gpt.img.xz

Abschließend wird das Debian-Image über den Loader in den eMMC Flash geschrieben:

	sudo rkdeveloptool wl 0 radxa-cm3-io-debian-bullseye-xfce4-arm64-20221101-0302-gpt.img

... und der SoC neu gestartet:

	sudo rkdeveloptool rd

Nach dem Neustart sollte auf einem angesteckten HDMI Bildschirm der Bootprozess sichtbar sein. Der "Boot" Schalter auf dem IO Board kann nun wieder auf "off" gestellt werden. 

Verbindet man das IO Board via Ethernet mit dem lokalen Netzwerk, ist das Radxa Modul über den Hostnamen `radxa-cm3-io` erreichbar:

	ssh rock@radxa-cm3-io

Standard-Logindaten sind

* Username: `rock`
* Password: `rock`


## APT aktualisieren

Direkt nach der Installation konnten wir in unserem konkreten Fall keine Paketupdates für Debian einspielen, da Uhrzeit und Datum noch nicht korrekt eingestellt waren. Das ließ sich mittels 

	sudo systemctl restart ntp.service

schnell beheben. 

Zudem wurde mindestens ein APT Repository nicht also gültig erkannt, weil ein PGP-Schlüssel nicht aktuell war:

	$ sudo apt update && apt upgrade -y
	
	W: GPG error: http://apt.radxa.com/bullseye-stable bullseye InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 9B98116C9AA302C7

Auch dieses Problem ließ sich mit einem einzigen Kommando schnell beseitigen: 

	wget -O - apt.radxa.com/$(lsb_release -c -s)-stable/public.key | sudo apt-key add -

Danach funktionierte auch ein `sudo apt update` wieder einwandfrei. 



