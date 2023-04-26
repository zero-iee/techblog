---
title: "Waveshare 4 Inch display does not work with IO BASE MODULE B"
date: 2023-04-26T14:47:13+02:00
draft: false
author: "Thomas Leister<thomas.leister@zero-iee.com>"
tags: ["waveshare", "raspberrypi", "cm4", "display"]
---

Because we had to find out the painful way: The [Waveshare IO BASE Module B](https://www.waveshare.com/wiki/CM4-IO-BASE-B) for the Raspberry Pi CM4 module does *not* work with the [4" DSI Touch Display](https://www.waveshare.com/wiki/4inch_DSI_LCD) from Waveshare - at least not as long as you use a BASE IO board revision < 4. 

Only from board version 4 the higher performance DSI1 interface is used instead of the DSI0 interface of the Raspberry CM4 by the IO Base Board. 

The information comes from Waveshare support, which we contacted because of our problems with the display. The change of the DSI port with version 4 of the IO Base Board is [documented](https://www.waveshare.com/wiki/CM4-IO-BASE-B#Version_Introduction), but unfortunately there is no reference to the missing compatibility to the display anywhere at the moment - so it is documented here ... ;-)
