---
title: "Embedded Linux: Video in Endlosschleife auf Framebuffer abspielen mit mplayer"
date: 2022-12-21T02:38:26-08:00
draft: false
author: "Thomas Leister <thomas.leister@zero-iee.com>"
tags: ["linux", "embedded", "video", "mplayer", "ubuntu"]
---

Vor allem für Demozwecke, z.B. auf Messen, fragen Kunden immer wieder nach Displayansteuerungen, die eine Videodatei auf einem oder mehreren Bildschirmen präsentieren. Während sich zum Teil abenteuerliche Lösungen mit Windows und automatisch startenden PowerPoint-Präsentationen mit eingebettetem Video finden lassen, waren wir von ZERO auf der Suche nach einer eleganteren Lösung, die zudem zügig startet und wenig Raum für Fehlbedienung lässt. 

<!--more-->

Zentraler Bestandteil in unserem Ubuntu-basierten Setup ist der `mplayer` - ein Videoplayer, der seinen Videoutput nicht nur in einer grafischen Desktopumgebung einbetten kann, sondern diesen auch ganz ohne GUI direkt auf den Framebuffer schreiben kann. So können wir auf das Laden einer Xorg- oder Wayland-basierten Desktupumgebung verzichten und beschlenigen den Bootprozess deutlich. Zudem entstehen keine Hürden wie Login oder automatisch startende Applikationen, die das Video verdecken (z.B. der Ubuntu Softwareupdatedienst). 

Der `mplayer` kann aus den Standardpaketquellen von Ubuntu bezogen werden:

    apt update
    apt install mplayer

Nach der Installation wird unter `/opt/runvideo.sh` ein Script angelegt, das `mplayer` mit einigen Parametern startet:

```
#!/bin/bash
VIDEOFILE=$(find /home/showdisplay/Videos/ -type f  -print -quit)
mplayer -vo sdl:driver=fbcon ${VIDEOFILE} -loop 0
```

In der zweiten Zeile wird die erste Videodatei ermittelt, die sich im Verzeichnis `/home/showdisplay/Videos` befindet (nach alphabetischer Reihenfolge). In Zeile 3 passiert dann die eigentliche `mplayer`-Magie:

* `-vo sdl:driver=fbcon`: Video auf dem Framebuffer wiedergeben
* `-loop 0`: Das Video in einer Endlosschleife abspielen. **WICHTIG: Dieser Parameter muss an letzter Stelle stehen - sonst startet nicht nur das Video neu, sondern die gesamte `mplayer` Instanz. Das führt zu einem deutlichen Flackern beim Neustart. Das Verhalten ist nicht dokumentiert. Deshalb sei hiermit explizit darauf hingewiesen.**

Das Startscript wird nun noch ausführbar gemacht: 

    chmod u+x /opt/runvideo.sh

... und eine `systemd` Servicedatei wird erstellt, um das Script beim Boot automatisch auszuführen:

```
[Unit]
Description=Run video
ConditionPathExists=/opt/runvideo.sh

[Service]
Type=forking
ExecStart=/opt/runvideo.sh
TimeoutSec=0
StandardOutput=tty
RemainAfterExit=yes
SysVStartPriority=99

[Install]
WantedBy=multi-user.target
```

Zuletzt wird der neue Service aktiviert: 

    systemctl daemon-reload
    systemctl enable runvideo.service


Da keine GUI benötigt wird (bzw. in diesem Fall sogar stört), wird der Gnome Desktop Manager (GDM) deaktiviert:

    sudo systemctl disable gdm


Fertig!

Sobald der Displaycontroller neu gestartet wird, wird die erste Videodatei aus `/home/showdisplay/Videos/` abgespielt - zumindest, solange das Format vom `mplayer` unterstützt wird. 


