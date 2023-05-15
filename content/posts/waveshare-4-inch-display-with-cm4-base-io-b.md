---
title: "Waveshare 4 Inch Display funktioniert nicht mit Waveshare CM4 Base IO Module B"
date: 2023-04-26T14:33:50+02:00
draft: false
author: "Thomas Leister"
tags: ["waveshare", "cm", "raspberrypi", "display"]
---


Weil wir es auf die schmerzhafte Weise herausfinden mussten: Das [Waveshare IO BASE Module B](https://www.waveshare.com/wiki/CM4-IO-BASE-B) für das Raspberry Pi CM4 Modul funktioniert *nicht* mit dem [4" DSI Touch Display](https://www.waveshare.com/wiki/4inch_DSI_LCD) von Waveshare - zumindest nicht, solange man eine BASE IO Board Revision < 4 verwendet. 

Erst ab Board Version 4 wird die höherperformante DSI1 Schnittstelle statt der DSI0 Schnittstelle des Raspberry CM4 vom IO Base Board genutzt. 

Die Information stammt vom Waveshare-Support, den wir wegen unserer Probleme mit den Display kontaktiert haben. Die Änderung des DSI-Ports mit Version 4 des IO Base Boards ist zwar [dokumentiert](https://www.waveshare.com/wiki/CM4-IO-BASE-B#Version_Introduction), doch leider befindet sich zum aktuellen Zeitpunkt nirgendwo ein Hinweis auf die fehlende Kompatibilität zum Display - daher sei es hier dokumentiert ... ;-)

