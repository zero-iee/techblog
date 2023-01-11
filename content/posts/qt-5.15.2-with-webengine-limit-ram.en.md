---
title: "Qt 5.15.2 with WebEngine (Chromium) - Limit RAM usage to avoid crashes"
date: 2023-01-11T14:19:06+01:00
draft: false
author: "Thomas Leister <thomas.leister@zero-iee.com>"
tags: ["qt", "chromium"]
---


When compiling Qt 5.15.2 from the official open source sources, we encountered a problem in combination with our build server: The build process was interrupted while compiling the Chromium-based "WebEngine" component with initially mysterious error messages. A look at the kernel log using `dmesg -w` then quickly revealed that the so-called OOM killer of the Linux kernel had struck. Apparently the RAM consumption of the build process was so memory-intensive that the process had to be aborted to keep the operating system running. 

But how could this be? Our build server has 32 GB of RAM and 24 CPU cores. A quite powerful machine. It should not actually reach its performance limits so quickly.

The problem is caused by two factors. First, building the Chromium browser engine is extremely memory intensive. Generally, [not less than 16 GB RAM is recommended](https://chromium.googlesource.com/chromium/src/+/main/docs/linux/build_instructions.md). In our case, however, there is another problem: By default, "Ninja" - the build system used in Chromium - creates a build thread for each available virtual CPU core, so that the build process is parallelized to the maximum. What may still work well for a standard PC with 16 GB RAM, however, forces our build server with its 24 cores to its knees. Every single thread needs a not to be underestimated amount of RAM. In the end, the ratio of CPU cores and available RAM is no longer correct on our server, so the build process stops. 

The problem can be prevented if we artificially reduce the number of Ninja threads to be used - if we don't build with 24 CPU cores, for example, but only with 18. 

For this purpose the environment variable `NINJAJOBS` can be set before a `make -j$(nproc)`. Contrary to what one might expect _(and contrary to what is described in the [LFS manual](https://www.linuxfromscratch.org/blfs/view/svn/x/qtwebengine.html))_, however, not just a number is stored here, but the complete `-j` [parameter of Ninja](https://manpages.debian.org/testing/ninja-build/ninja.1.en.html):

	export NINJAJOBS="-j16"
	
If a `make` is subsequently executed, the usual Qt components are compiled with all cores, while the Ninja-based parts (in this case Chromium as part of the WebEngine) are built with fewer CPU kernels to conserve RAM. 

For our combination of 32 GB RAM and 24 CPU cores, we experimentally determined a count of 16 kernels with which to still run our build process. With only 8 CPU cores, RAM usage peaked at about 12 GB. 



