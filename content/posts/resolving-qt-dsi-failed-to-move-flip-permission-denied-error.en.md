---
title: "Prevent DSI output errors when starting a Qt application (Raspberry Pi)"
date: 2023-09-26T09:39:39+02:00
draft: false
author: "Thomas Leister"
tags: ["qt", "raspberrypi", "hdmi", "dsi", "boot"]
---

Yesterday, while completing some of our AMPS units with display and Qt application, we encountered an error that caused the application to fail to launch correctly and crash in rare cases. The error looks like this:

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

"In rare cases", because the bug was apparently timing-dependent. However, because we were not completely unaware of the error messages, we were able to find out relatively quickly where they were coming from: 

Our application runs in EGLFS mode and claims complete control over the display. There is no XServer, Wayland or similar running in the background. The error messages indicate a permission problem, but actually come from the fact that another application is already exerting control over the display. 

In our case: The boot splash screen (`plymouth`) of our Rasbian distribution. The error occurs only about 1-2 times out of 10, because in most cases systemd starts the application at a time when the display has already been released. Depending on the boot time - which can vary slightly - it can happen that the timing is so unfavorable that the splash screen has _not_ released the display when our application already wants to use it. 

Fortunately, this problem can be solved by adding another dependency to the systemd service of our application via `After`, e.g. 

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


_By the way: In this context, it may also be worthwhile to check whether the user running the application is in the "render" user group. The group membership is required in order to be allowed to access the display as an application via eglfs at all._