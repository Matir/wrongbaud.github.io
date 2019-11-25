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

### Dumping the Parallel flash with an ESP32

For this post, we're going to dump this flash using the ESP32 microcontroller, this is a very popular and well supported MCU that plenty of embedded peripherals as well as a wireless SoC that can be used for Bluetooth and WiFi comms. Below is a link to the develoment board we'll be using as well as a pinout.

https://www.espressif.com/sites/default/files/documentation/esp32-wroom-32_datasheet_en.pdf

ESP32.png

http://ww1.microchip.com/downloads/en/devicedoc/20001952c.pdf

In order to expand out IO capabilities to be able to interract with this flash chip we are going to use an I2C based IO expander chip called the MCP23017. This chip, as the name suggests, can be communicated with I2C and can be configured to read or write to 16 individual GPIO pins. This means, that be using 2 I2C pins on the ESP32 (SDA, SCL), we'll gain 16 IO lines. We can also put multiple MCP23017 chips on the same I2C bus meaning that we will be able to interract with 48 pins just using I2C on the ESP32! Before we get involved with how this particular chip works, we will provide a brief overview of I2C for the unfamiliar.

#### Understanding I2C

In previous posts, we discussed UART and SPI, these two protocols are somewhat limited when it comes to addressing mutliple devices at once. For example, UART has no selection or addressing capability, and SPI requires the CS line to be pulled low which will quickly start to eat up GPIO lines with using multiple SPI chips on the same board. 

I2C uses two lines for communication Serial Data (SDA) and Clock (SCL). I2C is similar to SPI in that is synchronous, with the sampling being based on the SCL signal. Where I2C really begins to set itself apart from SPI is the ability to address certain chips on the bus within the I2C message frame. A brief overview of the message structure can be seen below:

| Start Condition | Address Frame | R/W Bit | ACK/NACK | Data Frame | ACK/NACK | Data Frame 2 | ACK/NACK | Stop Bit |
| --------------- | ------------- | ------- | -------- | ---------- | -------- | ------------ | -------- | -------- | 
| 1 Bit (H->L Transition) | 7 or 10 bits | 1 bit | 1 bit | 8 bits | 1 bit | 8 bites | 1 bit | 1 bit |

The address for an I2C slave is typically controlled by assering IO lines on the chip. For example on the MCP23017 chip pins 15:17 control the lower bits of the address variable. 

The R/W bit (read / write) is fairly self explanatory, the usage of reads / writes are chip dependent. For example you may write to a register of an I2C based sensor in order to configure or initialize it, after that you would read the data from it. We'll need to read and write from the MCP23017 chips in order to properly dump the parallel flash chip so we will see uses of both shortly. 

See the example be low from the Sparkfun I2C tutorial for a better explanation!

https://cdn.sparkfun.com/assets/6/4/7/1/e/51ae0000ce395f645d000000.png

#### Interfacting with the MCP23017

The MCp23017 has a series of internal registers that are used to configure the state and direction of the 16 GPIO lines. 

#### Dumping the Flash using the MCP23017

### Analyzing the ROM

### Re-writing the parallel flash from the ESP32

### Conclusion
With this writeup, we've learned about multiplexers, I2C and how parallel flash chips work. I hope that this was useful for those that were interested and as always please reach out with any corrections or questions!
