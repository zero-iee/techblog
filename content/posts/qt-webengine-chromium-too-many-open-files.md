---
title: "Qt Webengine (Chromium): Too Many Open Files"
date: 2023-02-08T10:36:44+01:00
draft: false
author: "Thomas Leister"
tags: ["qt", "webengine", "chromium"]
---


Wie bereits in früheren Blogposts erwähnt, nutzen wir innerhalb einer Qt Quick Anwendung die Chromium-basierte Qt WebEngine, um Webinhalte darstellen zu lassen. Im Laufe der Entwicklung haben wir allerdings einen unerfreulichen Bug entdeckt, der unsere Anwendung nach einiger Zeit im laufenden Betrieb abstürzen lässt. Zunächst friert das Web-Fenster ein - etwas später folgt dann der Rest der Anwendung, bis die Anwendung schließlich beendet wird.

In diesem Beitrag stellen wir einen Workaround vor, der für uns funktioniert hat. 

<!--more-->

Zunächst galt es, den Ursprung für die Crashes zu ermitteln. Da die Anwendung neben dem WebEngine-Teil auch noch aus anderen, komplexeren Modulen besteht, fiel der Verdacht nicht gleich auf den WebEngine-Anteil. Viel wahrscheinlicher schien zunächst ein klassischer Memory Leak Bug zu sein. Also beispielsweise das neue Erstellen von C++ Objekten, ohne invalide oder nicht mehr benötigte Objekte hinterher zu entfernen und ggf. vorhandene Referenzen zu entfernen. 

Die Fehlermeldung im Qt Creator Log war:

```
[...] ERROR:broker_posix.cc(46) Received unexpected number of handles
[...] ERROR:platform_shared_memory_region_posix.cc(249) Creating shared memory in /dev/shm/.org.chromium.Chromium.Sf3dsf: Too many open files (24)
```

Hier findet sich schon ein erstes Indiz, dass die Qt WebEngine etwas damit zu tun haben könnte. Dieser Teil unserer Anwendung scheint zu viele File Handles zu öffnen. Der Linux-Kernel verfügt über einen Kontrollmechanismus, der Anwendungen in der Zahl ihrer offenen File Handles beschränkt, sodass Ressourcenprobleme durch Amok-laufende Prozesse unwahrscheinlicher werden. Über das `ulimit` Tool lassen sich die aktuell geltenden Einschränkungen einsehen (im Gültigkeitsbereich der Shell).

Für einzelne Prozesse können die geltenden Limits mittels `cat /proc/<PID>/limits` abgerufen werden. Über ein `ls -1 /proc/<PID>/fd | wc -l` wird die Anzahl der offenen File Handles für einen Prozess zurückgegeben.

Für unsere Anwendung lag die Zahl viel zu hoch. Langsam bei ca. 7.000 Handles beginnend entwickelte sich die Zahl der offenen Handles extrem schnell. Nach wenigen Minuten lag sie bereits im sechsstelligen Bereich. 

Je nach Benutzung der Webanwendung stieg die Zahl kaum oder sehr schnell. Vor allem bei intensiverer Benutzung des Web-Anteils der Anwendung stieg die Zahl rasant an. Da unsere WebApp eine beträchtliche Anzahl von Ajax-Requests im Hintergrund an einen REST API Server schickt, fiel der Verdacht zunächst auf offene Verbindungen seitens der Webanwendung. Hatte Chromium einen Bug und hielt Verbindungen aus irgendeinem Grund offen? 

Nach Deaktivierung der meisten Anfragen war schnell klar, dass die Hintergrundanfragen die Zahl der offenen File Handles nicht bedeutend beeinflusste. Alles andere wäre auch tatsächlich ein schwerer Bug in Chrome gewesen. Doch wieso beeinflusste die Benutzung der Webanwendung die Anzahl der offenen Handles dennoch so stark?
  
Durch weiteres Experimentieren fanden wir heraus, dass unser Javascript-Code innerhalb der WebApp nichts mit dem Phänomen zu tun hatte. Stattdessen erhöhte sich die Anzahl der File Handles mit jeder visuellen Änderung innerhalb der WebEngine. Änderte sich eine Zahl oder wurde eine Animation aktiviert, schoss der Zähler in die Höhe. Änderte sich nichts auf der dargestellten Website, änderten sich auch die File Handles kaum. 

Hiermit waren wir mit unserem Code also raus. Der Bug lang im Verantwortungsbereich von Chromium, Qt oder Grafiktreiber-Code. Abschließend konnten wir die Ursache mangels Einsicht in die Funktionsweise der involvierten Komponenten nicht klären - Aber nach einigen weiteren Experimenten gelang es uns, unsere Anwendung so anzupassen, dass das Problem nicht mehr auftritt.

Der Schlüssel war am Ende, in der Qt-Anwendung folgende Einstellung zu setzen:

```
QCoreApplication::setAttribute(Qt::AA_UseSoftwareOpenGL);
```

Eigentlich bewirkt die [Einstellung](https://doc.qt.io/qt-6/qt.html#ApplicationAttribute-enum), dass eine softwaregestützte, alternative OpenGL Implementierung genutzt wird. In den meisten Fällen wohl keine besonders attraktive Option. In unserem Fall bewahrt sie uns aber vor den Problemen, die wir mit der Standardeinstellung hatten und verursacht keine weiteren relevanten Nebeneffekte. Die Performance ist für unseren Anwendungsfall okay, sodass wir hiermit leben können. 

Als Ursache bleibt eine Inkompatibilität oder ein Bug zwischen unserem Raspberry Pi 4 Grafiktreiber und/oder Chromium bzw. der Qt WebEngine zu vermuten. Wie sich schon an anderer Stelle herausgestellt hat, kann der Raspberry Pi Grafikstack verwirrend und in bestimmten Konstellationen fehlerbehaftet sein. Das ist allerdings ein "Rabbit hole" für ein andermal ... 
