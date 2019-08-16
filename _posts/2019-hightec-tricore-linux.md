---
layout: post
title:  "Tricore Basics: Using the Hightec Tricore Toolchain in Linux"
image: ''
date:   2019-08-4 00:20:10
tags:
- embedded
- tools
- automotive
description: ''
categories:
- embedded
- automotive
---

# Background

The Tricore CPU architecture is commonly found in automotive embedded systems, often running an RTOS or even just bare metal firmware. This post will go over setting up an entry level toolchain for the Tricore architecture under Linux, and how we can use this toolchain when reverse engineering automotive platforms. We will also go over and provide a very simple bare metal loader.

This will be the first in a series of posts about the Tricore architecture and reverse engineering Tricore based embedded systems. For this series we'll be targeting a Tricore 1767, which is an older but commonly used CPU in automotive systems. The goal at the end of these posts will be to get our own code running on an ECU in Bootloader mode.

If we want to be able to run our own code on an ECU we're going to have to need a toolchain first, which is where I'm choosing to start. In the past I've written all of my Tricore code in ASM and never really explored the idea of using any available compilers. This series will explore these options and hopefully help some people out along the way...or at least entertain!

# Goals

1. Setup HighTec Embedded toolchain under Linux (Ubuntu 18.04) using WINE.
2. Build a "Hello World" binary to toggle GPIO Lines and set up the Processor, while using linker scripts to ensure that our binary be loaded at a specific address (we'll discuss this in a later post)
3. Disassemble/Decompile the binary using GHIDRA

## Setting up the toolchain under WINE

Unzip the toolchain

```
unzip free_tricore_entry_tool_chain.zip
```

Run ```Setup.exe``` with WINE:

```
wine Setup.exe
```

Next click through the install as seen below

![Tricore Install](TRICORE_1.png)


Once the installation has finished, you will find the files we care about in the resulting directory:

```
~/.wine/drive_c/HIGHTEC/toolchains/tricore/v4.9.1.0-infineon-2.0/bin
```

You can add this to your path with the following:

```
export PATH=$PATH:~/.wine/drive_c/HIGHTEC/toolchains/tricore/v4.9.1.0-infineon-2.0/bin
```

Now try invoking the gcc compiler with ```wine``` as seen below:

```
wrongbaud@wubuntu:~$ wine tricore-gcc.exe
tricore-gcc.exe: fatal error: no input files
compilation terminated.
```

Great! So now lets right some simple code to toggle a GPIO line!

## Build a "Hello World" binary to toggle a GPIO line

Lets start with a simple main.c that will (according to our datasheet) initialize and toggle a GPIO line!

```
/*
 * main.c
 *  Created on: Mar 6, 2019
 *      Author: wrongbaud
 */

#include <tc1767.h>
#include <tc1767/scu/addr.h>
#include <tc1767/scu/types.h>
#include <tc1767/port5-struct.h>

typedef struct
{
	unsigned int _con0;
	unsigned int _con1;
}WdtCon_t;

void toggle_pin_13()
{
	unsigned int * GPIO_CTRL_REG = 0xF0000F1C;
	unsigned int * GPIO_OUTPUT_REG = 0xF0000F04;
	*GPIO_CTRL_REG = 0x80208020;
	*GPIO_OUTPUT_REG = 0x20002000;
}

void disable_gpios()
{
	unsigned int * GPIO_CTRL_REG = 0xF0000F1C;
	unsigned int * GPIO_OUTPUT_REG = 0xF0000F04;
	*GPIO_CTRL_REG = 0x80208020;
	*GPIO_OUTPUT_REG = 0x00000000;
}
void toggle_pin_15()
{
	unsigned int * GPIO_CTRL_REG = 0xF0000F1C;
	unsigned int * GPIO_OUTPUT_REG = 0xF0000F04;
	*GPIO_CTRL_REG = 0x80208020;
	*GPIO_OUTPUT_REG = 0x80008000;
}

void main()
{
	unsigned int wcon0,wcon1;
	volatile WdtCon_t *wdtaddr = 0xF00005F0;
	wcon0 = wdtaddr->_con0;
	wcon1 = wdtaddr->_con1;
	// Unlock WDT and disable ENDINT protection so we can reconfigure clocks
	wcon0 &= 0xffffff03;
	wcon0 |= 0xf0;
	wcon0 |= (wcon1 & 0xc);
	wcon0 ^= 0x2;
	wdtaddr->_con0 = wcon0;
	wcon0 &= 0xfffffff0;
	wcon0 |= 0x2;
	wdtaddr->_con0 = wcon0;
	(void) wdtaddr->_con0;
	// Reconfigure clock speed as reccomended in user manual
	SCU_PLLCON0_t * SCU_PLLCON0 = (SCU_PLLCON0_t*) SCU_PLLCON0_ADDR;
	SCU_PLLCON0->bits.VCOBYP = 0;
	SCU_PLLCON1_t * SCU_PLLCON1 = (SCU_PLLCON1_t *) SCU_PLLCON1_ADDR;
	SCU_PLLCON1->bits.K1DIV = 0;
	toggle_pin_13();
	toggle_pin_15();
}
```

Next, we'll try to compile this code as follows:

```
wine ~/.wine/drive_c/HIGHTEC/toolchains/tricore/v4.9.1.0-infineon-2.0/bin/tricore-gcc.exe main.c
```

Oh no! The first big mistep in our journey has appeared!

```
wrongbaud@wubuntu:~/blog/tricore$ wine tricore-gcc.exe hello.c
license check: Can't read license data (-102)
No such file or directory (errno: 2)
license check: No valid license!
hello.c:1:0: error: error in licenser

 ^
hello.c:1:0: error: license check failed
```
Remember the License file we needed earlier? It looks like wine / tricore-gcc.exe can't find it. After a fair amount of time digging through docs, and troubleshooting wine, I loaded the tricore-gcc.exe in IDA and found out that there was a crucial argument not displayed when you run ```tricore-gcc.exe --help``` ...

The argument ```-mlicense-key=dir``` can be used to specify a directory containing the license file, so lets copy that into our working directory and run the following...

```
wine ~/.wine/drive_c/HIGHTEC/toolchains/tricore/v4.9.1.0-infineon-2.0/bin/tricore-gcc.exe hello.c -mlicense-dir=$(pwd) 
```

And we have a resulting ```a.out``` in our directory!

So at this point, we've installed the HIGHTEC toolchain under WINE on Linux and can compile binaries for the TC1767 CPU, now our eventual goal is to develop code that will be used as a bootstrap loader and our target will not likely be able to load ELF files, so we'll need to write some linker scripts in order to make sure that the resulting ```.text``` section of the binary can be extracted and ran on the target. We will cover this in the next blog post!

Initially when putting this post together, my plan was to use IDA-Pro to disassemble the binary and analyze what we've written to determine how efficient this HIGHTEC compiler is. The reason that I want to investiage this is that later on we will write our own Bootstrap code that will run on a target and we will want it to be as minimal as possible. 

## Disassemble/Decompile the binary using GHIDRA

In case you've not installed it yet, pull down the latest version of GHIDRA from: 


### Helpful Links

* Tricore Example Code / Loaders (will be useful for later posts)
  * http://www.infineon-autoeco.com/BBS/Detail/6835
  * http://bbs.21ic.com/archiver/tid-148660.html
  * https://github.com/w01230/inf_tc1791_bootloader
