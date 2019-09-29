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

With this post our goal will be to extract the firmware from the platform and locate and type of debugging if possible (UART,JTAG,etc). We will explore multiple ways of attempting to extract the filesystem and outline the steps taken for each method. 

**Disclaimer:** I've done my best to not research this router on the internet, so I apologize if this is old information for those who look at these platforms. 

## Teardown

Below is a picture of the PCB after removing it from the case:

![Router PCB Board](https://wrongbaud.github.io/assets/img/ROUTER_PCB_COLORED.png)

You can see the various SoCs highlighted  in red/yellow/green and the SRAM highlighted in blue, otherwise this is a very straightforward board. 

The CPU is highlighted in yellow, it has the part number: QCA9563-AL3A, this is going to be the main CPU that we interract with. 

The chip highlighted in red has the part number QCA9880, a quick google search returns the following: https://wikidevi.com/wiki/AIRETOS_AEX-QCA9880-NX 

The SoC highlighted in green has the part number QCA8337N which is described in it's datasheet as "highly integrated seven-port Gigabit Ethernet switch"

On the underside of the board there is a serial EEPROM which looks strikingly similar to those that we extracted on the Arcade cabinets in the previous posts


### Dumping the SPI flash

As in previous posts, this platform contains a Winbond SPI chip. Knowing what we learned about flashrom, we can hook up a buspirate and attempt get a dump of the flash. The flash chip in question is the Winbone W25Q16Fw, the datasheet can be found here: https://www.winbond.com/resource-files/w25q16fw%20reve%2010072015%20sfdp.pdf

Using the pinout from the diagram, we wire up a TSOP8 clip and connect it to the buspirate, and below are the results:

```
FLASHROM OUTPUT
```

It looks like we're having some issues doing this in circuit, looking at the lines with a scope, it appears that the BusPirate can not drive the lines appropriately. This is a common issue with trying to do any kind of in circuit reads, in order to mitigate this one can look for a reset line for the main CPU, or attempt to get the CPU into a state where it is still powering the chip but not attempting to communicate. But -- since I'm a little lazy, let's just remove it with hot air and dump it, see the pictures below and the output of binwalk on the flash image!

So we've succesfully dumped the flash, mainly due to me being stubborn and wanting to demonstrate some soldering, but as most of you probbly know, these firmware images are typically obtainable online which we will discuss next!

### Finding firmware online

When looking at routers such as this, it's often very easy to find firmware online. I wanted to attempt to extract the SPI flash because that's more in my wheelhouse being an embedded systems guy. However - the simpler path would be to go to the following link where we can find an update image! However, the ability to re-write the SPI flash can be useful in case we manage to brick the platform in the future!

Sure enough, googling the router name takes us to the following link where the firmware can be downloaded: https://static.tp-link.com/2019/201908/20190827/Archer%20A7(US)_V5_190810.zip

One should note however, that these firmware images contain the root filesystem of the target and not the bootloader, which is something we would be interested in if we were to attempt to port another OS or image to this platform potentially, so there is _some_ validation for dumping the SPI flash!

### Looking For Serial

When looking at an embedded system of this class for the first time one of the first things you want to look for is a serial port, these can lead to additional  debug information and in some cases even terminal access. Debug ports are often exposed via a UART and typically consist of a transmit (Tx) and receive (Rx) line. 

Of course some systems only give access to Tx and other have no access at all, but it's stil a good first step to try to hunt these down when assessing an embedded system. Methods of discovery for serial terminals can vary based on the system but there are a few things that can lead to quick wins:

1. Debug headers or vias
2. Silk screen labeling
3. Unused pads / jumpers on the PCB

So what do headers look like? What are vias? Below are some example images of what they typically look like and after looking at these you'll notice something very similar on our board!

![EXAMPLE_1](https://wrongbaud.github.io/assets/img/CENT_BOARD.jpeg)

![EXAMPLE_1](https://wrongbaud.github.io/assets/img/CENT_BOARD.png)


Looking ar our board we can see some headers that look very similar, now we need to inspect them, but first it might help to learn a little more about what a UART actually is!

![SERIAL_1](https://wrongbaud.github.io/assets/img/SERIAL_1.jpg)

#### UWAT? What is a UART?

UART stands for Univeral Ansynchronout Receiver Transmitter, this can be used for many things on an embedded system including communicating with other processors, sensor communications and of course what we're interested in - debug access. One of the features of using a UART over something like SPI or I2C is that since it is asynchronous, there is no clock to synchronize the signals being sent between the two. In place of using a clock the transmit line utilizes start and stop bits to outline a data packet. This means that both the transmit and recieve line must know the rate at which to read bits off of the wire as they come across, this is referred to as the baud rate. This is essentially the speed of the data being transferred which is typically measured in bits per second. 

So if we were to look at a UART packet, we would have the following

| Location | Name | Description | 
| -------- | ---- | ----------- | 
| 0:1 | Start bit | Used to signify the start of a packet | 
| 1:9 | Data bits (this can also be configured to be any value really, but is commonly 8) | The data to be send / read, note that data is typically sent with the least significat bit first | 
| 9:10 | Parity bit | This bit determines if data has been changed as it went over the wire | 
| 10:12 | Stop bits | This signifies that the packet has ended | 

This is all great information, but in practice what does it mean for us reverse engineers? Well depending on the logic level that the UART is using, we can probe the circuit board with a multimeter and look for fluctuations in the voltage, this might lead us to a debug console that is spewing log messages, or it may just be another bus being used for sensor communications. Let's start taking a look at the headers we outlined below and measure the voltages at each pad after the router has booted. 

| Pin | Value | 
| --- | ----- |
|  1  |   3.3V   | 
|  2  |   0 (GND)   | 
|  3  |   0   | 
|  4  |   3.3V   |

The Tx line is typically held high in most reference designs by whoever is driving it, AKA transmitting. We have 2 possible candidates for Tx. Pins 1-3. There is a good chance that one of these is our 3.3V VCC line, we can determine that by noticing the pin's behaiour when the platform is powered down. If it is a fast drop to 0V we can assume that it is a GPIO line of some sort, if it slowly drops down to 0V, it is likely the VCC line because that slow drop is caused by capacitors that are often present on power rails! After looking at these pins on shutdown we can assume that pin 1 is VCC, leaving one candidate for standard Tx, pin 4! 

Also note that we talked about how it's often up to the transmitter to drive the line high, which would mean that the Rx line (whic we would Transmit to) is likely pin 3 because it's 0V and not connected to ground!

When we look at this under the scope - there is no traffic or fluctuation whatsoever. However, take a second look at the photo and noice the pads labeled ```R24``` these pads indicate that a SMD resistor was here at some point in time and removed pre-production. Often times when looking at things like serial ports or JTAG connectors, there will be signs of resistors or other components being removed as these features are no longer needed once the device is in production. Let's probe the left side of that pad and see what it looks like with the logic analyzer.

![LOGIC_1](https://wrongbaud.github.io/assets/img/LOGIC_1.jpg)

Excellent! This is a serial terminal that drops us to a root shell, we could not have asked for more in this scenario. Let's take a look at the full serial dump:

```
SERIAL DUMP GOES HERE
```

In order to make this a more usable shell, I applied a solder blob bridging the two pads of the R24 connector. Note that some times in this scenario that one of the pads may be pulling the serial line up to the proper voltage, but since our voltage levels look normal on the multimeter and on the logic analyzer we can just use a solder blob. Now the header contains a functioning serial port!

## Analyzing the FS Image / SPI Dump

I showed before the binwalk output of the SPI dump. This showed us that on the SPI flash we have the UBoot image (Stage 2 bootloader) the kernel and the rootfs. This is everything we need to start poking around at the files running on the platform. 

## Next Steps
Now that we have the filesystem image and an active console we can focus on setting up a debug environment and hunting for any potential issues that may be present on the platform!

#### Image References
* https://harryskon.wordpress.com/category/reverse-engineering/
* http://thecablemodemguide.tripod.com/page38.htm
