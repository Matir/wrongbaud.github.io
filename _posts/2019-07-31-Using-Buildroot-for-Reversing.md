---
layout: post
title:  "Using Buildroot when Reverse Engineering"
image: ''
date:   2019-07-30 00:06:31
tags:
- embedded
- linux
- reversing
description: ''
categories:
- linux
---

# Using Buildroot when Reverse Engineering

## Overview

When reverse engineering an embedded system that is Linux based, one often wishes that they had an examplar system that could be virtualized, if only to gain familiarity with the nuances of the specific kernel version or to learn more about the running applications without needing the native hardware. Make no mistake, this is a bit of a pipe dream when working with bespoke embedded systems, but if you're working with a more generalized system (or if you just want to quickly spin up a Linux system to test against using QEMU) (Buildroot)[Buildroot.link] can be used to generate images that will run in QEMU! The purpose of this post will be to describe how to build specific kernel images for QEMU using buildroot.

## So what is Buildroot?
Buildroot is a collection of tools that can be used to build all of the necessary components to get Linux running on various embedded platforms.  Using buildroot we can generate the following (and almost certainly more since I am no expert)

* Kernel Images
* Root Filesystem images, including busybox, etc
* Ramdisk images
* Probably a lot more!

The idea behind buildroot is that given a supported hardware target (or QEMU) one can build a specific version of the Linux kernel as well as a root filesystem image. Both of these can be customized to the user's liking but for the purposes of this post we're just going to build a version of Linux 3.10 that will run in QEMU because that is the kernel version of the current target that I am researching at the moment!

## Installing

## Building your first image

## Run it in QEMU!
```

$ qemu-system-arm \
  -M versatilepb \
  -kernel zImage \
  -dtb versatile-pb.dtb \
  -drive file=rootfs.ext2 \
  -append "root=/dev/sda console=ttyAMA0,115200" \
  -serial stdio \
  -net nic,model=rtl8139 -net user \
  -redir tcp:2222::22 \
```
## Building specialized images

So this is a problem! We want to build a 3.10 kernel, however the oldest GCC toolchain that we can use to build the kernel with is GCC 5.0 which is _too NEW_ for our target. So we're going to need to find a way to set up our own toolchain and point buildroot to it if we want to move further with using GCC. 

*Note:* If you want to use the uClib toolchain, that will in fact work perfectly fine in this scenario and you will not have to set up an external GCC toolchain, if you're trying to get something done quickly I'd reccomend this route!

## Conclusion

