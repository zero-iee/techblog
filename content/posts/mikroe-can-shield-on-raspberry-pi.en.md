---
title: "Using a Mikroe CAN SPI Click 3v3 Module on a Raspberry Pi 4"
date: 2023-06-20T14:31:26+02:00
draft: false
author: "Thomas Leister"
tags: ["can", "raspberrypi", "mikroe", "click"]
---

Mikroelektronika (MikroE) from Serbia offers within its "Click" ecosystem numerous function modules that can be operated on microcontrollers. One of them is the "[CAN SPI Click 3.3V](https://www.mikroe.com/can-spi-33v-click)" CAN controller module, which can be connected via the standardized "MikroBus" interface or SPI. To make the connection to a Raspberry Pi 4 work, we added the "[Pi 4 Click Shield](https://www.mikroe.com/pi-4-click-shield)". 

In this post we will briefly explain how we got the CAN module working in conjunction with the Raspberry Pi 4. 

<!--more-->


The CAN SPI Click 3.3V module contains according to [product description](https://www.mikroe.com/can-spi-33v-click) a MCP2515 CAN controller from Microchip. So that the chip can be recognized by its driver, the device tree must be adapted by means of an overlay. A suitable overlay is already on the Raspi stand distribution "Raspberry Pi OS", so that the suitable overlay must be activated only. 

For this the configuration file `/boot/config.txt` is adapted:

	[all]
	dtoverlay=mcp2515-can0,oscillator=10000000,interrupt=6

Important are the two parameters "oscillator" and "interrupt":

* `oscillator` describes the clock frequency of the clock on the CAN module. [The schematic](https://download.mikroe.com/documents/add-on-boards/click/canspi-33v/can-spi-click-33v-manual-v100.pdf) of the module reveals that it is a 10 MHz crystal - so 10,000,000 Hz must be specified here. 
* `interrupt` describes the GPIO pin of the Raspberry Pi, which transmits an interrupt signal on incoming CAN frames. The INT signal from just mentioned schematic can be traced further through the MicroBus interface to the Pi 4 Click Shield, where it is finally [connected to GPIO6 of the Raspberry Pi](https://download.mikroe.com/documents/add-on-boards/click/pi_4_click_shield/pi-4-click-shield-click-schematic-v101.pdf). Therefore the value "6" has to be specified here.

Continue with the configuration of the CAN network interface. In the new file `/etc/network/interfaces.d/can0` the following content is entered:

	auto can0
        iface can0 can static
        bitrate 500000

This sets a bitrate of 500 kBit/s on the CAN interface and starts the interface on boot. 

After a reboot of the Raspberry Pi you should see the following in the kernel messages:

	pi@raspi:~ $ dmesg | grep mcp
	[    8.239956] mcp251x spi0.0 can0: MCP2515 successfully initialized.

So the CAN controller has been detected successfully.

Also an `ip link` should now show a CAN interface with the status "UP":

	pi@raspi:~ $ ip link
	1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
	    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
	2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
	    link/ether e4:5f:01:af:96:bc brd ff:ff:ff:ff:ff:ff
	3: wlan0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DORMANT group default qlen 1000
	    link/ether e4:5f:01:af:96:bd brd ff:ff:ff:ff:ff:ff
	4: can0: <NOARP,UP,LOWER_UP,ECHO> mtu 16 qdisc pfifo_fast state UP mode DEFAULT group default qlen 10
	    link/can

To test the function we can make use of the `can-utils` and send a message to the Raspi itself via CAN:

	sudo apt install can-utils

In an SSH session the `candump` command is started to show all incoming CAN messages:

	sudo candump can0

In another SSH session, test data is sent via `cansend`:

	sudo cansend 123#FEFE

If the message just sent is also visible in the candump window, the CAN bus is working properly.

---

Sources: 

* https://download.mikroe.com/documents/add-on-boards/click/pi_4_click_shield/pi-4-click-shield-click-schematic-v101.pdf
* https://download.mikroe.com/documents/add-on-boards/click/canspi-33v/can-spi-click-33v-manual-v100.pdf
* https://download.mikroe.com/documents/datasheets/MCP2515.pdf
* https://crycode.de/can-bus-am-raspberry-pi
* https://www.beyondlogic.org/adding-can-controller-area-network-to-the-raspberry-pi/
