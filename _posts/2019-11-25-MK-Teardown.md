---
layout: post
title:  "Dumping a Parallel Flash with an ESP32"
image: ''
date:   2019-05-06 00:06:31
tags:
- hardware
- reversing
description: 'Using an ESP32 microcontroller to access and dump a parallel flash chip present on a BasicFUN Mortal Kombat cabinet'
categories:
- hardware
- reversing
---

# BasicFUN MK Teardown

## Background
I noticed not too long ago that a new BasicFUN cabinet came out featuring one of my favorite childhood games: Mortal Kombat. This of coursed piqued my interest and I decided to purchase one and perform a teardown and hopefully dump the flash!

One thing that is important to note is that unlike the previous cabinets that we looked at that were based on consoles that contained 6502 CPUs, the current opinion of people on the forums are saying that this is the Sega Genesis port of Mortal Kombat, meaning that if we manage to extract the storage of this platform we can disassemble some 68k code! The Sega Genesis was my only console growing up and I've reversed many Megadrive/32x ROMs - so needless to say I was excited to extract some data from this platform.

### Initial Teardown

Taking a look at the PCB, you'll notice that something is _very_ different than the other boards. Instead of an 8 pin SPI flash, we have a much larger chip with many more pins. This is a parallel flash chip, and it operates in a different manner than the SPI flash chips that we've extracted previously, lets start by explaining how parallel flash chips work.

boardlayout.jpg

## Understanding Parallel Flash
Parallel flash chips are very different than the storage mediums we've looked at in the past. They have individual address and data lines that are used to access certain addresses on the chip. See the pinout below from the relevant datasheet for this chip:

flash.jpg

Let's start by pointing out the pins that are most important for doing a simple chip readout accoridng to the datasheet:

Unrelated: I would highly reccomend you read along with the datasheet open, having the ability to properly read and interpret a datasheet is very useful when reversing hardware or doing embedded design! I will only be covering a few aspects about how these chips work with this post and there are a lot more features that you may need at some point!

| Pin Name | Purpose | 
| -------- | ------- |
| OE:Output Enable | This pin is asserted when we want data to be output from the chip | 
| CE: Chip Enable | This pin is used to control the internal state machine of the flash |
| WE: Write Enable | This pin is used when a write operation is performed |
| BYTE: Byte mode | This pin is used in to enable the reading of individual bytes, otherwise two bytes are read and written at a time |
| A0:A21 / Address Pins | These pins are all used to determine what address to read or write to based on the state of the flash chip | 
| D0:D16 / Data Pins | These are used to input or output data based on the state of the chip | 

In order to perform a read operation, the following things must be done in order

1. Write the address to be read on the address lines A0:AX
2. Pull CE (chip enable) to GND
3. Pull OE (output enable) to GND
4. Read the bytes (or byte depending on the state of the BYTE pin) on the D0:D16 lines

Note that all of these things have to be done within an appropriate time window that is outlined in the diagram below from the datasheet:

TIMING_DIAGRAM

Alright so we have a rough outline of how to perform read operations on the chip, nothing horribly complicated, aside from the amount of pins needed! So next you're probably wondering, "How on earth are we going to extract information from this chip when we need so many data lines!?" Well, luckily for us there are a number of ways that we can use external ICs (integrated circuits) to expand the amount of IO pins that we can use!

### Understanding I2C

### Dumping the Parallel flash with an ESP32

### Analyzing the ROM

### Re-writing the parallel flash from the ESP32

### Conclusion
With this writeup, we've learned about multiplexers, I2C and how parallel flash chips work. I hope that this was useful for those that were interested and as always please reach out with any corrections or questions!
