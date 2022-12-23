---
title: "Linux Kernelmodule mit Source-Änderungen neu kompilieren"
date: 2022-12-23T09:20:52+01:00
draft: false
author: "Thomas Leister <thomas.leister@zero-iee.com>"
tags: ["linux", "kernel", "kompilieren"]
---

Wer - wie wir bei der ZERO GmbH - mit neuer Hardware hantiert, die vom Linux-Kernel noch gar nicht oder nur teilweise unterstützt wird, muss in einigen Fällen bestehende Kerneltreiber anpassen. In unserem konkreten Fall ging es dabei um ein 5G Mobilfunkmodem, das vom uns eingesetzten Linux-Kernel noch nicht korrekt als solches erkannt wurde. Die Hardware war mit ihrer USB Vendor- und Product-ID noch nicht in den dazugehörigen Treibern registriert und weitere Detailanpassungen mussten in den betroffenen Subsystemen vorgenommen werden.

Glücklicherweise lagert die Linuxdistribution Ubuntu die meisten Kerneltreiber in flexibel generierbare Kernelmodule (`.ko` Dateien) aus, sodass nicht der gesamte Kernel nach Änderungen neu kompiliert werden muss. Wie Änderungen beispielhaft am USB-Treiber "`option`" durchgeführt werden können, wird im Folgenden erklärt. 

<!--more-->

Zunächst einmal wird eine Quelltextkopie des aktuell laufenden Linux-Kernels heruntergeladen. Unter Ubuntu funktioniert das ganz einfach über die Installation des jeweiligen `linux-source` Pakets:
    
    sudo apt install linux-source

Die Sourcen liegen nach der Installation unter `/usr/src/linux-source-5.15.0.tar.bz2` (Ubuntu 22.04.1) und können in das Home-Directory extrahiert werden:

    tar -xf /usr/src/linux-source-5.15.0.tar.bz2 -C ~/

Außerdem werden die Linux Header installiert - passend zur Kernelversion:

    sudo apt install linux-headers-$(uname -r)

Damit der Linux-Kernel bzw. seine Module überhaupt gebaut oder konfiguriert werden können, müssen ggf. noch einige weitere Softwarepakete installiert werden:

    sudo apt install linux-headers-$(uname -r)
    sudo apt install build-essential libncurses-dev gawk flex bison libssl-dev dkms libelf-dev libudev-dev libpci-dev libiberty-dev autoconf llvm

Nach einem Wechsel in das entpackte Source-Verzeichnis wird die Konfiguration des aktuell laufenden Kernels in die Buildumgebung übertragen:

    cd ~/linux-source-5.15.0/
    make oldconfig

Bevor es an die Änderungen im Sourcecode der Kerneltreiber geht, wird die Buildumgebung noch vorbereitet:

    make scripts prepare modules_prepare

An dieser Stelle können nun die nötigen Änderungen am Treiber vorgenommen werden. In unserem Beispiel also unter `drivers/usb/serial/option.c`. Sobald die Änderungen gespeichert sind, kann das betroffene Modul oder Subsystem (in diesem Fall `usb/serial`) neu gebaut werden:

    make -C /lib/modules/$(uname -r)/build M=$(pwd)/drivers/usb/serial

Dabei werden in `drivers/usb/serial` neue `.ko` Dateien generiert, die nun in das System installiert werden können:

    sudo cp --backup drivers/usb/serial/option.ko /lib/modules/$(uname -r)/kernel/drivers/usb/serial/option.ko

Ein abschließendes

    sudo depmod

generiert die Kernelmodul-Abhängigkeiten neu, sodass nach einem Neustart des Systems die veränderte Version des Kernelmoduls/-treibers geladen wird.

Beachtet, dass diese Methode, Änderungen ins System einzupflegen, nicht unbedingt "Update-resistent" ist. Da wir hier "am Paketmanager vorbei arbeiten", ist sich dieser unserer Änderungen nicht bewusst und überschreibt ggf. unsere Modulversion mit einer neuen, von Canonical ausgelieferten Version. Bei Updates des Kernels oder seiner Module ist also Vorsicht geboten. Für kleine proof-of-concepts eignet sich dieser Weg dennoch. 