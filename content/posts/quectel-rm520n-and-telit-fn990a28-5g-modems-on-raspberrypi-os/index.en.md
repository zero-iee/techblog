---
title: "Using Quectel RM520M and Telit FM990A28 5G Modem with Raspberry Pi OS"
date: 2023-05-31T12:44:58+02:00
draft: false
author: "Thomas Leister"
tags: ["5g", "connectivity", "raspberrypi", "quectel", "telit", "cellular"]
---

On our odyssey in search of a 5G cellular modem, we have tried several modems from different manufacturers. Unfortunately, the commissioning was not always successful. Sometimes the driver support in the Linux kernel was completely missing - sometimes the control via NetworkManager / ModemManager was buggy or not possible at all. 

Easy commissioning and stable operation are important to us. Since we do not want to use the modems on only a few devices, a manual adaptation of the Linux kernel is usually out of the question for us. The effort involved is too great and the consequences for the further life cycle of a product are too unclear. Therefore, the operating system - often a Raspberry Pi OS - should be used in its factory state and without major adjustments, if possible. 

**Two modems have emerged for us that work "out of the box" in combination with the current [Raspberry Pi OS](https://www.raspberrypi.com/software/operating-systems/) on Debian 11 "Bullseye" basis (kernel 6.1):**

<!--more--> 

* **[Quectel RM520N](https://www.quectel.com/product/5g-rm520n-gl)**
* **[Telit FM990A28](https://www.telit.com/devices/fn990axx/)**

both 5G modems support not only the currently widespread "5G New Radio" with LTE Control Plane (NSA), but also 5G NR Standalone (SA), so that with a corresponding expansion stage of the 5G mobile network can also benefit from extremely lower latencies. 


## Hardware Setup


Out hardware setup is as follows:

* Raspberry Pi CM4 
* [Waveshare Dual Ethernet IoT Base Board](https://www.waveshare.com/wiki/CM4-DUAL-ETH-4G/5G-BASE) (with M.2 Slot)
* Telit FM990A28 M.2 Modul _or_
* Quectel RM520N 
* IoT SIM card of Deutsche Telekom


{{< figure src="images/waveshare-board.jpg" title="Waveshare Board with Quectel 5G Modem" >}}


## Commissioning 

The commissioning of our Telit module went as follows: _(similar for Quectel!)_

Check the visibility of the module in the USB subsystem:

	$ lsusb

A Telit Device should be visible here: 

	[...]
	Bus 002 Device 003: ID 1bc7:1070 Telit Wireless Solutions FN990
	[...]

The ModemManager should also recognize the module:

	tom@raspberry:~ $ mmcli -L
    /org/freedesktop/ModemManager1/Modem/0 [Telit] FN990A28

Using

	mmcli -m 0

some details about the modem can be output - among others, if there is a connection to the mobile network or if a SIM card is assigned. 

In our first attempt, a red `sim-missing` was displayed in the "Status" section, although a SIM card was inserted in the slot of the Waveshare Base IO module. Also attempts with other SIM cards were of no use - no card was detected in the system. 

### Fix "SIM Missing" problem

_(This problem does not occur with the Quectel modem!)_

A look into the [Schematics of the Waveshare Board](https://www.waveshare.com/w/upload/4/46/CM4-DUAL-ETH-4G_5G-BASE_SchDoc.pdf) revealed that the signal line ("CD" - "Card detect") for the physical detection of a SIM card in the slot is not continued to the M.2 slot, so that the cellular module _cannot_ detect a corresponding signal. No wonder, then, that we were permanently shown a "**sim-missing**".

{{< figure src="images/waveshare-schematics.png" title="Screenshot of the Waveshare Dual Ethernet IoT Base Board" >}}

The problem can be fixed by configuring the modem to accept a SIM card that is always inserted and to stop performing HotSwap queries. The configuration is done via AT commands within a terminal session with the modem itself:

	sudo apt install minicom
	sudo minicom -D /dev/ttyUSB2

Issue AT commands - should be acknowledged with "OK".

	AT#HSEN=0,0
	AT#HSEN=0,1

_(more precisely, this disables HotSwap for both potential SIM slots supported by the modem)_.

The Minicom session can be terminated with CTRL-A followed by "X".

For the change to be applied, the Raspi including the modem was rebooted / the power supply was interrupted. 

After a reboot the SIM card was finally recognized in the ModemManager - at the end of the output of `mmcli -m 0` a "SIM" line with the D-Bus path to the SIM device was displayed:

	SIM | dbus path: /org/freedesktop/ModemManager1/SIM/0


**The just mentioned adjustment at the Telit modem is not necessary at the Quectel modem!**

The next step is to activate the respective modem:

	mmcli -m 0 --enable


### Setting up a connection using NetworkManager

A new cellular connection is now set up using NetworkManager. To do this, we create a new "Connection" in the NetworkManager. In the background, this communicates with the ModemManager in order to pass on APN details to it.

For our telecom card, the following APN information is to be used:

* APN: internet.telekom
* IP-Type: ipv4
* Username: telekom
* Password: tm

The APN information of each provider can be looked up quickly on the Internet.

Unfortunately, we had no luck with the newer IPv6-enabled APN of Telekom `internet.v6.telekom` (and "ipv4v6") - we could not establish a connection. Problems in the IPv6 stack of the modem drivers are already known to us. Possibly they come to bear here as well. Therefore we are content with pure IPv4 support for the time being.

Before the connection can be established, it must be ensured that the NetworkManager is running: 

	systemctl enable --now NetworkManager

On a stock Raspberry Pi OS image this is not the case. A reboot after the systemctl command can't hurt. In our case, the interaction between the two managers only worked after a reboot. 

Finally, this command creates a new GSM connection in the NetworkManager:

	mmcli c add type gsm ifname cdc-wdm0 con-name telekom apn internet.telekom connection.autoconnect yes

In the case of Telekom, the APN "internet.telekom" including the other parameters is already stored in the SIM profile, so that only the name of the matching APN profile needs to be specified. User name and password can usually be omitted. 

If this does not work, other parameters can be specified as an alternative, e.g. 

	mmcli c add type gsm ifname cdc-wdm0 con-name telekom apn internet.telekom gsm.username telekom gsm.password tm gsm.pin 1234 connection.autoconnect yes

Especially the specification of a `gsm.pin` is important if the SIM card is protected with a PIN. Our SIM is not protected with a PIN, so the specification is omitted.

A `nmcli c` should now show that a new connection "telekom" has been created. If the connection is marked green, the login to the network has worked:

{{< figure src="images/networkmanager-ok.png" title="nmcli c" >}}

The ModemManager should also look similar to this after a short while, showing a "connected" in the status: _(`mmcli -m 0`)_

{{< figure src="images/modemmanager-ok.png" title="mmcli -m 0" >}}

### Check function

Whether the cellular connection is actually working can be quickly and easily checked by pinging the cellular device:

	ping -I wwan0 1.1.1.1

Latency typically hovers around >= 25 ms on a 5G NSA network, but can vary greatly. We have observed latencies as high as 600 ms - depending on reception and network load. 

When rebooting the Raspberry Pi, the cellular connection is automatically reestablished. 

By the way: An `ip route` reveals that NetworkManager has created a default route for the 5G modem. However, since this has a metric of 700, the cellular modem is only used if a destination cannot be reached over a possible Ethernet connection. So an `apt update` and everything else should normally run over an available Ethernet connection. If this is not available, the cellular connection serves as a fallback (hence the `-I wwan0` parameter in the `ping` command - this forces a connection via cellular).
