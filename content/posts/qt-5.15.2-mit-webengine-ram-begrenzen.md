---
title: "Qt 5.15.2 mit WebEngine (Chromium) bauen - RAM-Verbrauch begrenzen"
date: 2023-01-11T14:19:06+01:00
draft: false
author: "Thomas Leister <thomas.leister@zero-iee.com>"
tags: ["qt", "buildprozess", "chromium"]
---


Beim Kompilieren von Qt 5.15.2 aus den offiziellen Open Source Quellen sind wir in Kombination mit unserem Buildserver auf ein Problem getoßen: Der Buildprozess wurde  beim Kompilieren der Chromium-basierten "WebEngine" Komponente mit zunächst mysteriösen Fehlermeldungen unterbrochen. Ein Blick in das Kernellog mittels `dmesg -w` offenbarte dann schnell, dass der sog. OOM-Killer des Linux-Kernels zugeschlagen hatte. Offenbar war der RAM-Verbrauch des Buildprozesses so speicherintensiv, dass der Prozess abgebrochen werden musste, um das Betriebssystem lauffähig zu halten. 

Doch wie konnte das sein? Unser Buildserver verfügt über 32 GB RAM und 24 CPU-Kerne. Eine durchaus leistungsstarke Machine. Sie sollte eigentlich nicht so schnell an Leistungsgrenzen kommen. 

Das Problem wird durch zwei Faktoren verursacht. Zum einen ist das Bauen der Chromium Browser-Engine extrem speicherintensiv. Generell werden [nicht weniger als 16 GB RAM empfohlen](https://chromium.googlesource.com/chromium/src/+/main/docs/linux/build_instructions.md). In unserem Fall kommt aber noch ein weiteres Problem hinzu: Per default erstellt "Ninja" - das in Chromium eingesetzte Buildsystem - für jeden verfügbaren virtuellen CPU Core einen Build-Thread, damit der Buildprozess maximal parallelisiert wird. Was für einen haushaltsüblichen PC mit 16 GB RAM noch gut funktionieren mag, zwingt unseren Buildserver mit seinen 24 Cores allerdings in die Knie. Jeder einzelne Thread braucht eine nicht zu unterschätzende Menge RAM. Am Ende stimmt bei unserem Server das Verhältnis aus CPU-Cores und verfügbarem RAM nicht mehr, sodass der Buildvorgang abbricht. 

Das Problem lässt sich verhindern, wenn wir die Anzahl der zu nutzenden Ninja-Threads künstlich verkleinern - wenn wir also nicht mit 24 CPU-Kernen bauen, sondern beispielsweise nur mit 18. 

Dazu kann vor einem `make -j$(nproc)` die Umgebungsvariable `NINJAJOBS` gesetzt werden. Anders, als man vielleicht vermuten würde _(und anders, als es im [LFS-Handbuch](https://www.linuxfromscratch.org/blfs/view/svn/x/qtwebengine.html) beschrieben wird)_, wird hier allerdings nicht einfach nur eine Zahl hinterlegt, sondern der vollständige `-j` [Parameter von Ninja](https://manpages.debian.org/testing/ninja-build/ninja.1.en.html): 

	export NINJAJOBS="-j16"
	
Wird darauf folgend ein `make` ausgeführt, werden die üblichen Qt Komponenten mit allen Cores kompiliert, während die Ninja-basierten Anteile (in diesem Fall Chromium als Teil der WebEngine) mit weniger CPU Kernen gebaut werden, um den RAM zu schonen. 

Für unsere Kombination von 32 GB RAM und 24 CPU Cores haben wir experimentell eine Anzahl von 16 Kernel ermittelt, mit denen unser Buildprozess noch durchläuft. Mit nur 8 CPU-Kernen hatte die RAM-Auslastung bei ca 12 GB ihren Peak. 



