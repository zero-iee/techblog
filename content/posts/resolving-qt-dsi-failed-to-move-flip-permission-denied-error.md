---
title: "DSI Output Fehler beim Start einer Qt Anwendung verhindern (Raspberry Pi)"
date: 2023-09-26T09:39:39+02:00
draft: false
author: "Thomas Leister"
tags: ["qt", "raspberrypi", "hdmi", "dsi", "boot"]
---

Gestern haben wir bei der Fertigstellung einiger unserer AMPS Einheiten mit Display und Qt-Applikation einen Fehler festgestellt, der dazu geführt hat, dass die Anwendung in seltenen Fällen nicht korrekt gestartet werden kann und crasht. Der Fehler sieht wie folgt aus:

<!--more-->

```
-- Logs begin at Mon 2023-09-25 12:17:01 CEST, end at Mon 2023-09-25 12:32:43 CEST. --
    Sep 25 12:32:24 0601-010200-0012 systemd[1]: Started app.
    Sep 25 12:32:27 0601-010200-0012 tester[444]: Failed to move cursor on screen DSI1: -13
    Sep 25 12:32:27 0601-010200-0012 tester[444]: Failed to move cursor on screen DSI1: -13
    Sep 25 12:32:27 0601-010200-0012 tester[444]: Could not set cursor on screen DSI1: -13
    Sep 25 12:32:28 0601-010200-0012 tester[444]: Could not set DRM mode for screen DSI1 (Permission denied)
    Sep 25 12:32:28 0601-010200-0012 tester[444]: Could not queue DRM page flip on screen DSI1 (Permission denied)
    Sep 25 12:32:29 0601-010200-0012 tester[444]: Could not queue DRM page flip on screen DSI1 (Permission denied)
```

"In seltenen Fällen", weil der Bug offenbar Timing-abhängig war. Weil uns die Fehlermeldungen allerdings nicht ganz unbekannt waren, konnten wir relativ schnell herausfinden, woher sie kamen: 

Unsere Anwendung läuft im EGLFS Betrieb und beansprucht die vollständige Kontrolle über das Display. Im Hintergrund läuft kein XServer, Wayland oder ähnliches. Die Fehlermeldungen weisen auf ein Berechtigungsproblem hin, kommen aber eigentlich daher, dass bereits eine andere Anwendung Kontrolle über das Display ausübt. 

In unserem Fall: Der Boot-Splashscreen (`plymouth`) unserer Rasbian Distribution. Der Fehler tritt nur ca 1-2 von 10 Mal auf, weil systemd die Anwendung in den meisten Fällen zu einem Zeitpunkt startet, zu dem das Display bereits wieder freigegeben wurde. Je nach Bootzeit - und die kann ja bekanntlich leicht variieren - kann es aber auch passieren, dass das Timing so ungünstig ist, dass das Splashscreen das Display noch _nicht_ freigegeben hat, wenn unsere Anwendung dieses bereits nutzen will. 

Das Problem lässt sich glücklicherweise beseitigen, indem zum systemd Service unserer Anwendung mittels `After` eine weitere Abhängigkeit hinzugefügt, z.B. 

	[Unit]
	Description=App
	After=systemd-user-sessions.service plymouth-quit.service
	
	[Service]
	Type=simple
	User=pi  
	Group=pi
	ExecStart=/opt/tester/bin/tester -platform=eglfs
	
	[Install]
	WantedBy=multi-user.target



_Übrigens: In diesem Zusammenhang kann es sich auch lohnen, zu prüfen, ob der Benutzer, welcher die Applikation ausführt, in der Benutzergruppe "render" ist. Die Gruppenzugehörigkeit wird benötigt, um als Anwendung überhaupt via eglfs aus das Display zugreifen zu dürfen._