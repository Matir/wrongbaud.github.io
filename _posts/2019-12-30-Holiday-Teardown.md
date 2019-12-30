---
layout: post
title:  "BasicFUN Series Part 5: Holiday Hardware Teardown"
image: ''
date:   2019-12-30 00:06:31
tags:
- hardware
description: 'Hardware teardown of handheld electronics'
categories:
- hardware
- reversing
---

# Overview
Over the holiday break, I recieved a few more random game platforms from friends and family who know how much I enjoy tearing into these things. While I didn't find anything amazing or insightful, I did use some techniques and tools that I've not mentioned before here so I wanted to go over them in more detail. The purpose of this post is not neccesarily about what lies within these ROM files, but more on the methods and tools used to extract the information. 

## Target 1: BasicFUN Oregon Trail

First and foremost, I should say that lots of people have taken this apart and even started to dig through the ROM a little but, first who comes to mind is [@foone on twitter](https://twitter.com/Foone?s=20), as well as [Tom Nardi from hackaday](https://hackaday.com/2018/03/14/teardown-the-oregon-trail-handheld/). With that out of the way, lets take a look at the main board and see what we can identify.

![Oregon Trail Board](https://wrongbaud.github.io/assets/img/dec-teardown/oregon-trail-board.jpg)


Ok here we see the following part numbers on two TSOP8 chips that are placed in similar locations to the [other platforms](https://wrongbaud.github.io/BasicFUN-flashing/) we've looked at in the past. Googling these part numbers results in the following

* [I2C 24C04 EEPROM](https://www.endrich.com/fm/2/HT24LC08.pdf)
* [SPI GD25Q80 EEPROM](http://www.elm-tech.com/en/products/spi-flash-memory/gd25q80/gd25q80.pdf)

Excellent, one of these is a SPI flash which we have seen and dealt with before, while the other is an I2C based eeprom. We have covered how [I2C works in previous posts](https://wrongbaud.github.io/MK-Teardown/) but this will present us another opportunity to dig into this protocol with a different target.

### Dumping the Flash with an FT2232
In the previous posts we used a buspirate to extract the flash, which is what I have traditionally used. After chatting with [@joegrand on twitter](https://twitter.com/securelyfitz?s=20), I decided to give the [FT2232H](https://www.amazon.com/FTDI-Breakout-Board-Dual-Channel/dp/B06XGGGMB7) breakout boards a try to see how well they work in comparison. We will still be using [flashrom](https://github.com/flashrom/flashrom) for the extraction but we'll have to wire up the FT2232H accordingy.

**NOTE:** Despite many attempts and multiple soldering jobs, I was not able to get consistend readouts working in circuit, so for this platform we will be removing the SPI flash and placing it on a breakout board as seen below:

[BREAKOUT_BOARD]

With the breakout board wired up, we connect the following pins from the SPI flash chip to the FT2232H board

| Flash Pin | FT2232H Pin | 
| --------- | ----------- |
| ```CS```  | ```AD3``` | 
| ```MOSI``` | ```AD1 ```|
| ```MISO``` | ```AD2 ```| 
| ```CLK``` | ```AD0``` | 

With these wired up we will run flashrom as follows:

```
wrongbaud@wubuntu:~/blog/dec-teardown$ flashrom -p ft2232_spi:type=2232H -r oregon-trail-cab.bin
flashrom v1.1-rc1-125-g728062f on Linux 5.0.0-37-generic (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Using clock_gettime for delay loops (clk_id: 1, resolution: 1ns).
Found GigaDevice flash chip "GD25Q80(B)" (1024 kB, SPI) on ft2232_spi.
Reading flash... done.
```


The first thing that I noticed, which had been mentioned to me be @securelyfitz, is that this was _noticably_ faster than using the buspirate, this readout only took a matter of seconds!

It is important when extracting chips like this to always perform a few readouts to make sure that you're getting consistent data.

```
wrongbaud@wubuntu:~/blog/dec-teardown$ flashrom -p ft2232_spi:type=2232H -r oregon-trail-cab-2.bin
flashrom v1.1-rc1-125-g728062f on Linux 5.0.0-37-generic (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Using clock_gettime for delay loops (clk_id: 1, resolution: 1ns).
Found GigaDevice flash chip "GD25Q80(B)" (1024 kB, SPI) on ft2232_spi.
Reading flash... done.
wrongbaud@wubuntu:~/blog/dec-teardown$ diff oregon-trail-cab.bin oregon-trail-cab-2.bin
```

Great, it looks like we've got a good firmware image of this platform. Digging through some of it in a hex editor, we see a reference to a debug mode, not unlike what we saw in [OTHER_CABINETS]

### Debug Mode

If you hold a certain button combination while powering up this platform, the following debug menu appears:

![Oregon Trail Test Mode](https://wrongbaud.github.io/assets/img/dec-teardown/oregon-trail-test.jpg)


One of the interesting things on this menu is that there is the string ```24C04```...OK. What exactly is this doing? Presumably it is testing the I2C based EEPROM on the board, but what does this test look like?

In order to introspect on what this test is actually doing, let's hook up a cheap logic analyzer to the SDA/SCL pins of the EEPROM and enter the test mode again. Hopefully we will be able to see something come across the bus in Pulseview

![Pulseview](https://wrongbaud.github.io/assets/img/dec-teardown/pulseview.png)

Sure enough, we see I2C traffic, but what exactly is it doing? If you need a primer on I2C, please see my previous post for some more information.

If we look at the first few packets, we see the following:

![I2C Packet 1](https://wrongbaud.github.io/assets/img/dec-teardown/i2c-debug-1.png)
![I2C Packet 2](https://wrongbaud.github.io/assets/img/dec-teardown/i2c-debug-2.png)
![I2C Packet 3](https://wrongbaud.github.io/assets/img/dec-teardown/i2c-debug-3.png)

Ok so what exactly is happening here? First, 8 write operations are performs as shown in the images above. If we look at the datasheet, we can see that the following sequence outlines how to write to the flash device

![I2C Packet 3](https://wrongbaud.github.io/assets/img/dec-teardown/byte-write.png)

So the first 8 packets are writing bytes ```E1-E8``` to addresses ```F0-F7```. After these writes are performed, the test simple reads them back out to check that the proper bytes were written which can be seen in the screenshot below. Neat!

![I2C Test Read](https://wrongbaud.github.io/assets/img/dec-teardown/i2c-test-read.png)

Aside from this the debug menu has the standard tests for button presses and things like that. Moving on we'll take a look at the contents of the I2C EEPROM and talk about how to extract these types of devices.

### Exploring the I2C Flash

## Target 2: ATGames Blast 
### Hardware Overview
### Extracting the flash with Flashtality
### Exploring the firmware

## Conclusion
