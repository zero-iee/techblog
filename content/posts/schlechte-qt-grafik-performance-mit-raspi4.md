---
title: "Schlechte Qt Grafikperformance mit Raspi4"
date: 2023-02-07T08:19:47+01:00
draft: false
author: "Thomas Leister"
tags: ["qt", "raspi", "raspberrypi", ""]
---

Während der Entwicklung einer Qt-basierten Anwendung auf einem Raspberry Pi 4 (CM) sind wir von der ZERO GmbH vor ein paar Tagen auf ein kurioses Problem gestoßen: Die Anwendung beinhaltet zwei Anwendungsfenster - ein QT Quick Fenster und ein weiteres Fenster, das mittels WebEngine (Chromium) eine Webanwendung darstellt. Die Inhalte in beiden Fenstern wurden während des Debuggings und beim manuellen Starten der Binärdatei aus der Kommandozeile heraus flüssig dargestellt. Bei Animationen und Mausbewegungen konnten wir keine bemerkenswerten Ruckler feststellen. 

Doch nach einem Start während des Boot-Prozesses durch ein Systemd Service-File sah die Sache schon ganz anders aus: Die Darstellung ruckelte stark und die CPU-Auslastung lag bereits bei einfachen Texteinblendungen bei ca 80 %. 

Wir konnten einen Zusammenhang zwischen dem Startzeitpunkt der Anwendung und der Performance feststellen. In frühen Bootphasen war die Performance schlecht - bei späterer Ausführung lief die Anwendung flüssig. Durch ein "sleep" Kommando in unserem Systemd Service File konnten wir den Zusammenhang verdeutlichen:

```
[Unit]
Description=App

[Service]
User=pi
Group=pi
Type=simple
ExecStart=/bin/bash -c 'sleep 10; /opt/myapp/myapp -platform=eglfs'

[Install]
WantedBy=multi-user.target
```

Alleine schon eine Verzögerung der Ausführung um 10 Sekunden führte zum gewünschten Ergebnis. Dies war jedoch keine Zufriedenstellende Lösung - schließlich wollten wir die Qt-Anwendung so schnell wie möglich starten und nicht unnötig Zeit verlieren, ohne den Grund für das Verhalten zu kennen.

Ein Blick in das Systemlog offenbarte schließlich, dass unsere Anwendung den LLVMpipe Rasterizer nutze, wenn sie über ein Service-File gestartet wurde:

```
"Feb 03 04:33:29 raspberrypi bash[470]: Running on a software rasterizer (LLVMpipe), expect limited performance."
```

Letztendlich konnten wir das Performanceproblem nach einigen Websuchen durch ein Hinzufügen unseres `pi` Benutzers zur Benutzergruppe `render` (und einen Anschließenden Neustart) beheben:

```
sudo usermod -aG render pi
```

Nach [Aussage des Forennutzers _"cRaZy-biScuiT"_](https://www.hardwareluxx.de/community/threads/raspberry-pi-os-und-ein-paar-fehler-beim-%C3%B6ffnen-eines-programmes.1272595/post-27563953) wird damit ein Bug behoben, der dafür sorgt, dass Anwendungen unter dem `pi` User keine hardwarebeschleunigten Anwendungen ausführen können. 

Unsere Qt-Anwendung startete nun auch schon in früheren Bootphasen inkl. Hardwareunterstützung und die Performance war wie erwartet. 

Bisher können wir allerdings nicht erklären, wieso das Problem zu späteren Startzeitpunkten nicht auftrat. Wir vermuten eine externe Beeinflussung evtl. durch später startende, dritte Anwendungen, die dafür sorgte, dass die Hardwarebeschleunigung zu diesen Zeitpunkten dennoch zur Verfügung stand. 


