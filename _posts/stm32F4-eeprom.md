---
layout: post
title:  "Using an STM32 to assist with hardware RE"
image: ''
date:   2019-05-06 00:06:31
tags:
- random
description: ''
categories:
- hardware
- reversing
---


Similar CPU https://www.cnx-software.com/2017/12/03/sega-genesis-flashback-retro-game-console-is-powered-by-monkey-king-3-6-processor-runs-android/

The serial EEPROM can be found here: http://ww1.microchip.com/downloads/en/DeviceDoc/doc3256.pdf

All of the address pins are pulled low, meaning we can ignore the A[0-2] pins on the EEPROM.

Refs:
http://www.mind-dump.net/configuring-the-stm32f4-discovery-for-audio

Next steps: 
    * Read bootup / Capture with Salaea to get better picture
    * Probe for serial output with nice probes / record logic levels with Multimeter

