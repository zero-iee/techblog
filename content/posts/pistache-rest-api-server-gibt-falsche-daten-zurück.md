---
title: "Pistache REST API Server gibt falsche Antworten auf Anfragen zurück"
date: 2023-01-27T12:17:09+01:00
draft: false
author: "Thomas Leister"
tags: ["cpp", "pistache", "rest", "api", "javascript", "web"]
---


Bis vor wenigen Stunden hatten wir mit einem merkwürdigen Bug im Zusammenhang mit der C++ HTTP Library "[Pistache](https://pistacheio.github.io/pistache/)" zu tun, der sich zunächst nicht ganz identifizieren ließ. Möglicherweise sind wir nicht die einzigen Betroffenen - deshalb wollen wir in diesem Post kurz das Setup und unseren Fix vorstellen. 

<!--more-->

Die Umgebung besteht aus einem C++ basierten Backend, von welchem Daten über eine REST API ausgelesen und in einem Webbrowser dargestellt werden sollen. 

Die Abfrage der Daten von der API erfolgt dabei über ein Javascript heraus. Wir arbeiten ohne Library - ganz traditionell mittels [XMLHttpRequest](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest/Using_XMLHttpRequest). Da einige verschiedene Datensätze auf der Website dargestellt werden, werden im Hintergrund periodisch mehrere parallele [Ajax](https://de.wikipedia.org/wiki/Ajax_(Programmierung))-Anfragen an die REST API formuliert und übertragen. 

**Das Problem bestand nun darin, dass wir - scheinbar zufällig - immer wieder Ajax-Antworten zurück bekamen, die wir in diesem Kontext gar nicht angefordert hatten.** Wurde zum Beispiel eine Anfrage nach allen verfügbaren Autos abgeschickt, bekamen wir die Antwort für die Nachfrage nach allen verfügbaren Schiffen. Parallel wurden im Hintergrund zwar auch alle verfügbaren Schiffe angefragt - aber eben nicht in _der_ Funktion, die für die Autos zuständig war. Es schien, als würden die Antworten zu HTTP-Anfragen teilweise vermischt. 

Die erste Vermutung bestand darin, dass wir einen Fehler in unserem Javascript hatten und bei gleichzeitigen Anfragen Variablen miteinander überschrieben. Nach sorgfältiger Überprüfung und eindeutiger Umbenennung der involvierten Variablen konnten wir das aber ausschließen. 

Ein Bug im Webbrowser, der dafür sorgte, dass Anfragen und Antworten vermischt wurden? Unwahrscheinlich. Das Problem trat sowohl im Chromium- als auch im Firefox Webbrowser auf. 

Dann musste es am Backend liegen. Wir begannen, das Backend genauer zu untersuchen. Dabei stellte sich heraus, dass die Probleme immer nur dann auftraten, wenn eine gewisste HTTP-Handler-Funktion aufgerufen wurde. Diese eigene Funktion wird von der Pistache Library aufgerufen, wenn eine Anfrage eingeht. Innerhalb der Funktion können die Parameter der Anfrage geprüft und verarbeitet sowie eine passende Antwort formuliert werden. 

Durch schrittweises Auskommentieren innerhalb der Funktion und Reduzierung auf das Wesentliche (nämlich das Senden einer passenden Antwort an den Webbrowser) konnten wir das Problem schließlich eingrenzen.

Innerhalb der Funktion gab es folgenden Code-Abschnitt: 

```
void ApiHandler::getVehicle(const Rest::Request &request, Http::ResponseWriter response){
    json j;
    [...]

    if (myModel->getType() == "car") {
    	[...]
        j["licensePlate"] = car->getLicensePlate();
        j["owner"] = car->getOwnerName();
        response.send(Http::Code::Ok, j.dump() + '\n'); // Respond with JSON string
    } else if (myModel->getType() == "ship") {
        [...]
        j["homeCountry"] = car->getHomeCountry();
        j["owner"] = car->getOwnerName();
        response.send(Http::Code::Ok, j.dump() + '\n'); // Respond with JSON string
    } 
        
    response.send(Http::Code::Unprocessable_Entity);
}
```

Fehler gefunden? Ganz einfach: Die Intention war, einen "Unprocessable_Entity" Fehler zurückzugeben, wenn die Funktion für ein anderes Modell als ein "Car" oder "Ship" Modell ausgeführt wurde. Dabei wurde allerdings ein `else` vergessen. Richtig müsste es heißen: 

```
	else {
		response.send(Http::Code::Unprocessable_Entity);
	}
```

Das `else` hier wegzulassen ist in Fällen möglich, in denen mittels "return" die weitere Abarbeitung der Funktion gestoppt wird. Hier aber nicht - in unserem Fall führt der Fehler dazu, dass in den meisten Fällen 2x ein `response.send` ausgeführt wird. 

Damit scheint der Pistache HTTP Server nicht zurecht zu kommen und verhält sich _undefiniert_. Wir haben innerhalb der Pistache-Library nicht weiter nachgeforscht, allerdings erschien es uns erwähnenswert, dass sich die Library in so einem Fall unvorhersehbar verhält und offenbar sogar Antworten auf gleichzeitige HTTP-Anfragen vermischt. 

Wer also mit einem sich unkontrolliert verhaltenden Pistache Server kämpft, sollte seinen Code ggf. einmal auf doppelte response.send() Anweisungen prüfen. 