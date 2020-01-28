---
layout: post
title:  "Dumping Xbox Controller Firmware via Single Wire Debug"
image: ''
date:   2020-01-30 00:06:31
tags:
- hardware
- reversing
- jtag
description: ''
categories:
- hardware
- jtag
---

## Background
I was looking around my apartment for potential targets for my next post and was pleasantly surprised to find the following XBox One controller still in the packaging:

![controller_pic](https://pisces.bbystatic.com/image2/BestBuy_US/images/products/6362/6362974_sd.jpg;maxHeight=640;maxWidth=550)

I don't really play my XBox that much so I thought it might be interesting to tear down this controller and see what kind of information we could extract from it.

## Hardware Teardown

Opening up the case reveals the following PCB:

Note that there really isn't too much to see here, as the main chip is covered in epoxy. Luckily for us a lot of the test pads are labeled, but the labeled ones seem to be test points for various button presses, so there's nothing exciting there.

There is an IC labeled AK4961 towards the bottom of the board, but this is an audio codec chip. The datasheet can be found [here](https://www.digikey.com/product-detail/en/akm-semiconductor-inc/AK4951EN/974-1064-1-ND/5180415). This chip is a  low  power  24-bit  stereo  CODEC  with  a  microphone,  headphone  and  speaker amplifiers. 

If we look to the right of this however there is a small grouping of pads with _some_ silk screen labelling:

![debug_pads]()

So we see ```3V3```,```A13```,```A14```,```RES``` labeled in the silkscreen. This is worth taking a look at, and if you've read my previous post about the router teardown and discovering UARTs you may already have some ideas on how to proceed here. We'll start by measuring the voltage of each pin with a multimeter:

| Pin | Value | 
| --- | ----- | 
| ??? | 0 (GND) | 
| RES | 3.3V | 
| A14 | | 
| A13 | | 
| 3V3 | 3.3V | 

There was no fluctuation or modulation on RES, A14 or A13, so these must be for something else, but what? Given that one of the labels is ```RES``` (which likely stands for system reset) there is a good chance that there are JTAG or SWD headers. 

We can test if the ```RES``` pin actually resets the target by pulling it low with a 10k resistor (remember we're reversing things here and don't want to accidentally short something!). If you are not familiar with these types of headers or how a system reset pin typically works - they are often _active low_ meaning that they idle at a high value and have to be pulled low to be activated. So if we monitor the output of ```dmesg -w``` and toggle this line low with a 10k resistor, what do we see?

```
[ 2108.588884] usb 1-6.4: new full-speed USB device number 10 using xhci_hcd
[ 2108.691108] usb 1-6.4: New USB device found, idVendor=0e6f, idProduct=02a2, bcdDevice= 1.0f
[ 2108.691113] usb 1-6.4: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[ 2108.691116] usb 1-6.4: Product: PDP Wired Controller for Xbox One - Crimson Red
[ 2108.691119] usb 1-6.4: Manufacturer: Performance Designed Products
[ 2108.691122] usb 1-6.4: SerialNumber: 0000AE38D7650465
[ 2108.698675] input: Generic X-Box pad as /devices/pci0000:00/0000:00:14.0/usb1/1-6/1-6.4/1-6.4:1.0/input/input25
[ 2131.403862] usb 1-6.4: USB disconnect, device number 10
[ 2133.420350] usb 1-6.4: new full-speed USB device number 11 using xhci_hcd
[ 2133.522469] usb 1-6.4: New USB device found, idVendor=0e6f, idProduct=02a2, bcdDevice= 1.0f
[ 2133.522474] usb 1-6.4: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[ 2133.522478] usb 1-6.4: Product: PDP Wired Controller for Xbox One - Crimson Red
[ 2133.522480] usb 1-6.4: Manufacturer: Performance Designed Products
[ 2133.522483] usb 1-6.4: SerialNumber: 0000AE38D7650465
[ 2133.530103] input: Generic X-Box pad as /devices/pci0000:00/0000:00:14.0/usb1/1-6/1-6.4/1-6.4:1.0/input/input26
```

Ah excellent, doing this caused the controller to reset, that's one pin down, 2 more to go.

When looking at debug headers like this, a common assumption is that it's for JTAG or some other form of hardware level debugging. However, the JTAG spec requires that there be at least 4 pins, TDO,TDI,TMS and TCK. We only have two on our target, so there is a good chance that this is a Single Wire Debug (SWD) port. 

## Understanding SWD
SWD is a common debugging interface that is used on popular SoCs like the STM32 series and many other microcontrollers. As the name states, SWD only requires one data line and one clock line, but how can we determine which one is which? Before we go down that route, we should understand a little more about how SWD works and what tools can be used to interface with it.

First off - SWD interfaces with something called a "Debug Access Port" (DAP). The DAP brokers access to various "Access Ports" (APs) which provide functionality including from typical hardware debugging, legacy JTAG cores, and other high performance memory busses. The image below pulled from [this document](https://stm32duinoforum.com/forum/files/pdf/Serial_Wire_Debug.pdf) provides a visual representation of how the DAP and APs are architected. 

![swd_arch.png]()

Each of these APs consist of 64, 32 bit registers, with one register that is used to identify the type of AP. The function and features of the AP determine how these registers are accessed and utilized. You can find all of the information regarding these transactions for some of the standard APs [here](https://static.docs.arm.com/ihi0031/c/IHI0031C_debug_interface_as.pdf)

#### SWD Protocol

As we mentioned before - SWD was developed as a pseudo-replacement for JTAG. With SWD the pin count was reduced from 4 to 2 and it provides a lot of the same functionality of JTAG. One downside to SWD however is that devices can not be daisy chained together, which JTAG allowed for. The two pins that are used in SWD are below:

| Pin | Purpose | 
| --- | ------- | 
| ```SWCLK``` | Clock signal to CPU, determining when data is sampled and sent on ```SWDIO``` | 
| ```SWDIO``` | Bi directional data pin used to transfer data to and from the target CPU | 

## Building OpenOCD
Install the dependencies:
```
sudo apt-get install build-essential libusb-1.0-0-dev automake libtool gdb-multiarch
```
Clone the repository, configure for our adpter (ST-Link V2), and build!
```
wrongbaud@115201:~/blog$ git clone https://git.code.sf.net/p/openocd/code openocd-code
cd openocd-code
./bootstrap
./configure --enable-stlink
make -j$(nproc)
```


## Refs
* https://electronics.stackexchange.com/questions/53571/jtag-vs-swd-debugging
* https://www.silabs.com/community/mcu/32-bit/knowledge-base.entry.html/2014/10/21/serial_wire_debugs-qKCT
