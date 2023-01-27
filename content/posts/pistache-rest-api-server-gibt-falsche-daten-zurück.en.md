---
title: "Pistache REST API Server Returns Incorrect Responses to Requests"
date: 2023-01-27T12:17:09+01:00
draft: false
author: "Thomas Leister <thomas.leister@zero-iee.com>"
tags: ["cpp", "pistache", "rest", "api", "javascript", "web"]
---


Until a few hours ago we had to deal with a strange bug related to the C++ HTTP library "[Pistache](https://pistacheio.github.io/pistache/)", which could not be identified completely at first. Maybe we are not the only ones - so in this post we want to briefly present the setup and our fix. 

<!--more-->

The environment consists of a C++ based backend from which data is to be read via a REST API and displayed in a web browser. 

The request of the data from the API is done via a Javascript. We work without a library - quite traditionally using [XMLHttpRequest](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest/Using_XMLHttpRequest). Since several different data sets are displayed on the website, several parallel [Ajax](https://de.wikipedia.org/wiki/Ajax_(programming)) requests to the REST API are periodically formulated and transmitted in the background.

**The problem was that we - seemingly randomly - kept getting Ajax responses back that we had not requested in this context.** For example, if a request for all available cars was sent, we got the response for the request for all available ships. In parallel, all available ships were also requested in the background - but just not in _the_ function that was responsible for the cars. It seemed that the answers to HTTP requests were partially mixed.

The first assumption was that we had a bug in our Javascript and were overwriting variables with each other on simultaneous requests. However, after careful checking and clearly renaming the variables involved, we were able to rule that out. 

A bug in the web browser that caused requests and responses to get mixed up? Unlikely. The problem occurred in both the Chromium and Firefox web browsers.  

Then it had to be the backend. We started to examine the backend more closely. It turned out that the problems only occurred when a certain HTTP handler function was called. This custom function is called by the Pistache library when a request is received. Within the function, the parameters of the request can be checked and processed, and a suitable response can be formulated. 

By gradually commenting out within the function and reducing it to the essentials (namely, sending a suitable response to the web browser), we were finally able to narrow down the problem.

Within the function there was the following code section: 

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

Found the mistake? Quite simple: The intention was to return an "Unprocessable_Entity" error if the function was executed for a model other than a "Car" or "Ship" model. However, an `else` was forgotten. Correctly it should be like this:

```
	else {
		response.send(Http::Code::Unprocessable_Entity);
	}
```

Omitting the `else` here is possible in cases where the further processing of the function is stopped by "return". But not here - in our case the error leads to `response.send` being run twice in most cases. 

The Pistache HTTP server does not seem to be able to cope with this and behaves _undefined_. We did not investigate further within the Pistache library, but it seemed worth mentioning that the library behaves unpredictably in such a case and apparently even mixes up responses to concurrent HTTP requests. 

So if you are struggling with an uncontrollably behaving Pistache server, you might want to check your code for duplicate response.send() statements. 