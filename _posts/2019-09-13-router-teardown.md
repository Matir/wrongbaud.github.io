---
layout: post
title:  "Router Analysis Part 1: Hardware Teardown"
image: ''
date:   2019-05-06 00:06:31
tags:
- hardware
- reversing
- router
description: ''
categories:
- hardware
- reversing
---
# Router Analysis Part 1: Teardown and Filesystem Analysis

## Overview

In previous posts, we've gone over how to tear down Arcade cabinets containing SPI Flash as well as how to dissect the data that was extracted from the Rom. With this next series of posts, I'd like to take the concepts we talked about on those platforms and demonstrate them on a more popular platform: The TP Link: $ROUTER_NUM. 

With this post our goal will be to extract the firmware from the platform and locate and type of debugging if possible (UART,JTAG,etc). 

**Disclaimer:** I've done my best to not research this router on the internet, so I apologize if this is old information for those who look at these platforms. 

## Teardown

Below is a picture of the PCB after removing it from the case:

ROUTER.png

You can see the various SoCs highlighted  in red/yellow/green and the SRAM highlighted in blue, otherwise this is a very straightforward board. 

The chip highlighted in yellow has the part number QCA9880, a quick google search returns the following: https://wikidevi.com/wiki/AIRETOS_AEX-QCA9880-NX

The SoC highlighted in green has the part number QCA8337N which is described in it's datasheet as "highly integrated seven-port Gigabit Ethernet switch"

On the underside of the board there is a serial EEPROM which looks strikingly similar to those that we extracted on the Arcade cabinets in the previous posts


### Dumping the SPI flash

### Analyzing the flash dump

### Finding firmware online
