---
title: "Quectel RM520N-GL 5G Mobilfunkmodem unter Ubuntu Linux in Betrieb nehmen"
date: 2023-01-02T14:20:04+01:00
draft: false
author: "Thomas Leister"
tags: ["linux", "ubuntu", "mobilfunk", "5g", "quectel"]
---

Um verschiedene 5G Usecases zu demonstrieren, sollte im Rahmen eines Projekts ein 5G-Modem an einem Ubuntu-basierten Mini PC betrieben werden. Hierzu haben wir uns das RM520N-GL von Quectel besorgt. 

Das Modem wird vom Linux-Kernel erst ab Version 6.0 vollständig unterstützt. Die dazugehörigen Patches sind [hier (USB "option" Treiber)](https://patchwork.kernel.org/project/linux-usb/patch/tencent_23054B863154DC02C6E98E5942416BFC200A@qq.com/) bzw [hier (qmi_wwan Treiber)](https://www.spinics.net/lists/linux-usb/msg230835.html) zu finden. 

Quectel liefert in der Dokumentation zwar auch Hinweise aus, an welchen Stellen die beteffenden Treiber angepasst werden müssen, doch das erwies sich in unserem Fall als fehleranfällig. Leider werden keine fertigen Git-Patches geliefert, sondern nur Code-Snippets in einem PDF-Dokument.

Einfacher war es, den Ubuntu 22.04.1 LTS Kernel von Version 5.17 auf Version 6.0.1 anzuheben. Dazu muss der Kernel und seine Module nicht unbedingt neu gebaut werden. Canonical stellt bereits vorkompilierte Kernelartefakte in Form von Debian-Paketen zur Verfügung. Diese werden zwar nicht offiziell unterstützt oder mit Updates versorgt, doch für unsere Demonstrationszwecke war das völlig ausreichend. 

Unter https://kernel.ubuntu.com/~kernel-ppa/mainline/v6.0.1/ werden vier wichtige .deb Pakete zum Download angeboten:

* `amd64/linux-headers-6.0.1-060001-generic_6.0.1-060001.202210120833_amd64.deb`
* `amd64/linux-headers-6.0.1-060001_6.0.1-060001.202210120833_all.deb`
* `amd64/linux-image-unsigned-6.0.1-060001-generic_6.0.1-060001.202210120833_amd64.deb`
* `amd64/linux-modules-6.0.1-060001-generic_6.0.1-060001.202210120833_amd64.deb`

Alle vier Dateien werden heruntergeladen und über den Paketmanager installiert:

    cd Downloads
    sudo dkpg -i ./linux*.deb

Target neu starten - und fertig! Ein `uname -r` sollte jetzt die neue kernelversion zeigen. Und auch die aktualisierten USB- und Netzwerktreiber für das Quectel Modem sollten jetzt den Dienst aufnehmen und die passenden Device Nodes unter /dev anlegen: 

* `/dev/cdc-wdm0`
* `/dev/ttyUSB0`
* `/dev/ttyUSB1`
* `/dev/ttyUSB2`
* `/dev/ttyUSB3`

_(USB Nodes können je nach angeschlossener Peripherie auch anders nummeriert sein!)_


## Eine Verbindung über den NetworkManager und ModemManager herstellen

Der "schönste" Weg führt über die NetworkManager-Integration des ModemManagers. Um zu sehen, ob das Quectel Modem überhaupt als solches vom ModemManager erkannt wurde, kann ein 

    mmcli --modem=0

ausgeführt werden. Ein paar Informationen zum Modemstatus sollten hier aufgelistet werden. 

Wer eine mit PIN gesicherte SIM-Karte im System nutzt, wird diese zuerst entsperren müssen. Dazu kann über das vorher genannte Kommando zuerst der virtuelle Pfad des SIM-Slots ermittelt werden, z.B.

    -----------------------------------
    SIM     |        primary sim path: /org/freedesktop/ModemManager1/SIM/0
            |        sim slot paths: slot 1: /org/freedesktop/ModemManager1/SIM/0 (active)
            |                        slot 2: none

... und dann die entsprechende SIM (Index 0) aktiviert werden: 

    mmcli -i=0 --pin=1234

Wer die SIM permanent entsperrt lassen will, kann die PIN-Sperre entfernen:

    mmcli -i=0 --pin=1234 --disable-pin

Sobald die SIM-Karte entsperrt ist, sollte eine Verbindung mit dem Modul möglich sein:

    nmcli c add type gsm ifname cdc-wdm0 con-name 5g apn internet.telekom
    nmcli c up 5g

Bis sich das Kommando zurückmeldet, können einige Sekunden vergehen. Wir haben hier absichtlich den alten `internet.telekom` APN gewählt, weil wir mit dem neueren "internet.v6.telekom" APN keine Verbindung herstellen konnten. Ursache (noch) unbekannt.


## Verbindung über das `qmi-network` tool

Statt des Network-/ModemManagers kann für die Verbindungsherstellung auch das `qmi-network` Script aus dem `libqmi-utils` Paket verwendet werden. Dazu werden in `/etc/qmi-network.conf` die richtigen Einstellungen für das Nodem hinterlegt:

    DEVICE=/dev/cdc-wdm0
    DEVICE_OPEN_QMI=YES
    PROXY=YES

Speziell die `DEVICE_OPEN_QMI` scheint hier eine wichtige Rolle zu spielen. Anders als in ähnlichen Anleitungen zur Inbetriebnahme von Mobilfunkmodems konnten wir hierauf nicht verzichten. 

Schließlich lässt sich eine Verbindung starten:

    sudo qmi-network /dev/cdc-wdm0 start

Allerdings kümmert sich das Script nicht um das Beziehen einer IP-Adresse und der entsprechenden Konfiguration des `wwan0` Netzwerkinterfaces. Hier hilft der `udhcpc` Client aus dem gleichnamigen Debian-Paket:

    sudo udhcpc -i wwan0

Im Idealfall ist nach wenigen Sekunden eine lokale IP-Adresse auf dem `wwan0` Interface konfiguriert und lässt eine Kommunikation ins Internet zu:

    curl ifconfig.me

zeigt dabei die IP-Adresse an, die für Verbindungen ins Internet aktuell genutzt wird. Es empfiehlt sich, vorher sicherzustellen, dass der Mini-PC über keine Ethernetverbindung ins Internet mehr verfügt. Dazu wird die default-Route zum lokalen Ethernetinterface `enp0s29f1` entfernt:

    sudo ip route del default via 192.168.179.1 dev enp0s29f1

_(kommando anpassen - je nach Output des Kommandos `ip route`)_





