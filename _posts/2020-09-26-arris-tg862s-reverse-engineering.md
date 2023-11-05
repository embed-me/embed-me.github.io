---
title: 'Reverse Engineering - ARRIS TG862S'
permalink: /arris-tg862s-reverse-engineering/
categories:
    - 'Reverse Engineering'
tags:
    - ARRIS
    - Hardware
    - 'Reverse Engineering'
    - SPI
    - TG862S
    - UART
---

## Intro

A couple of weeks ago, I found one of those old ARRIS TG862S Modems still lying around when cleaning up. I thought it might be nice to find another use-case for it, since it has been some time that the modem was actually in use with my former internet provider.

Once connected to power, it looked like the device booted properly, however, after a couple of minutes, the device seemed to be rebooting/crashing or whatever. The box was not reachable in the network and when looking a the LEDs of the ethernet ports, it seemed like it was stuck in a reboot loop. Since I have always been eager to learn something new and I am interested in how stuff works, I thought this would be a great opportunity to practice some reverse engineering in order to figure out what the problem was and how to fix it. Although I was not very successful with my attempt, this is what this blog post is about.

## Recon

As usual, first let’s try to find some information about the device using our favorite search engine. However, it seems like nobody else has ever tried to reverse engineer the device. Even though, there was some information on the Web, nothing that could help to fix my problem or provide useful insight (eg. leaked schematics, internal documentation…).

Furthermore, unfortunately ARRIS does not seem to provide a firmware download or similar for this device on their website.

## The Hardware

Seems like we have to open the device up. This is what the PCB looks like.

![](/assets/posts/tg862/NDP_1940.png)
![](/assets/posts/tg862/NDP_1939_small.png)

In order to get a rough overview, I tried to identify the main components/modules on the board.

![](/assets/posts/tg862/NDP_1940_smal_detailsl.png)

The PCBs copyright is 2012 and it seems to be revision T (whatever that means). Seems like the Board Version is 1.2. The CPU used is labeled DNCE2510GU from Intel, which seems to be a rebranded TI TNETC4800 (belonging to the Puma5 family) targeting cable modems. Thanks to mrquincle who has figured this out. Since I didn’t want to brick the device I kept all the shieldings in place and took a quick look at some of the other components. Let’s see:

- Power Supply (Blue)
- Multiport Gigabit Ethernet Switches (Yellow)
- Processor DNCE2510GU (Green)
- Flash MX25L12835F (Red)
- SDRAM M12L64164A (Brown)

## "Entry Point"

In order to figure out what caused the crash, USART access could give great insight. So let’s take a look. Knowing that the ports required are probably VCC, RX, TX and GND the pin header is an obvious candidate to investigate further. I used a simple UART to USB converter from China in order to interface the system. Start off finding GND and VCC using a multimeter, then connect RX of the Converter to one of the other pins and power up the system. If we are lucky, the developers have not disabled logging and we hit the TX pin of the chip with our first guess, we should see something on the terminal. We might need to adjust the baudrate, but in this case my initial guess of 115200 Baud already did the trick.


```console
foo@bar:~$ screen /dev/ttyUSB0 115200 1n8
```

![](/assets/posts/tg862/uart_jtag_pcb_small.png)

The first view lines on the terminal once connected were quite interesting and revealed some useful insight. The secondary bootloader is [Das U-Boot](https://u-boot.readthedocs.io/en/latest/), a very popular bootloader for embedded systems that I regularly use at work.

![](/assets/posts/tg862/uart_boot.PNG)

A couple of seconds after booted successfully, all of a sudden the system gets reset and from there on is stuck in a loop until a complete power cycle occurs. No information about what went wrong in the console, so that’s quite disappointing. However, because the reset happened out of nowhere, I suspected that it might be a watchdog timer that got triggered. Maybe because the arris application got stuck?

![](/assets/posts/tg862/uart_boot_reset.PNG)


## How about JTAG

Next, I tried to find the JTAG port of the DNCE2510GU in order to get some more insight. However, since the PCB has multiple layers identifying the individual ports without the use of a [JTAGulator](http://www.grandideastudio.com/jtagulator/) is not that ease. I quickly gave up, since at the current stage the information would not be that useful anyway.

## Getting some more insight

In order to get a better understanding of the system, let’s try to dump the content of the two flash chips. We have already identified that these are Micronix MX25L12835F, so let’s evaluate if your favorite tool flashrom can handle them. A quick look at the supported hardware section reveals that indeed it is. While using the Micronix datasheet to identify the pins, connect a SOP16 Pin Adapter to the chip on top and bottom. Finally, start to dump the EEPROM content.

![](/assets/posts/tg862/spi_flashdump.PNG)

Using binwalk in combination with the --extract parameter it is trivial to extract the rootfs from the EEPROM dump. Alternatively, it is always possible to use dd once the offset and size of the Squashfs are known.

![](/assets/posts/tg862/flashdump_top_binwalk.PNG)

After extraction, we have a full root filesystem at hand we can use to poke around. From here on depending on what we want to do, the opportunities are basically endless (eg. disassemble system binaries, find version information about services and check them for CVE‘s,…). However, for now stick to what is important, try to find the root cause of the reset and try to fix it.

![](/assets/posts/tg862/rootfilesystem.PNG)

Armed with more knowledge of the system, let’s get a rough overview of how the system is initialized.

![](/assets/posts/tg862/mini_cli.png)

Now that we have a rough overview of the boot process, let’s continue. First of all, let’s try to interrupt the boot-process in U-Boot. Unfortunately, this does not work, however, since the developers did not disable the serial port and boot-time is of the essence, I suspect that the bootdelay uboot environmental variable is set to zero. In order to still end up in the U-Boot console, I tried what happens if the CRC-Check of the kernel image fails. To achieve this I simply read from the external flash using a Bus Pirate device while the bootloader tried to read as well. This of cause resulted in a collision and the calculation of the CRC failed. A couple of seconds later I had access to the U-Boot console.

![](/assets/posts/tg862/uart_force_uboot.PNG)


Now, that we have full access to U-Boot we can do whatever we want, even tftpboot our own image. What is interesting is, that the reset behavior does not happen at this level. Maybe the root cause is really a watchdog activated at a later stage in the boot process? Let’s find out if we can get a shell. We already know that a normal boot process during a handoff in the kernel loads the busybox init process which in term runs rcS. In order to get what we want, we can simply pass init=/bin/bash to the kernel using the bootargs environmental variable.

```console
bootargs=root=/dev/mtdblock6 mtdparts=spansion:0x20000(U-Boot),0x10000(env1),0x10000(env2),0x7e0000@0x40000(UBFI1),0x7e0000@0x820000(UBFI2),0x1386c4@0x82193c(Kernel)ro,0x5ff000(RootFileSystem)ro;spansion1:0x60000@0xfa0000(nvram),0x7d0000@0(ARRS1),0x7cffc0@0x40(FS1),0x7d0000@0x7d0000(ARRS2),0x7cffc0@0x7d0040(FS2) console=ttyS0,115200n8 ethaddr0=00:00:CA:11:22:33 usbhostaddr= init=/bin/bash boardtype=tnetc550 eth0_mdio_phy_addr= ext_switch_reset_gpio=
```

Note that this has do be done for every reboot. The device uses a double buffering mechanism in order to switch between two images in case of an error (fallback) and makes use of the flash in order to store this information. However, saveenv does not work here, since a custom script always overwrites the bootargs environment variable with default values. I don’t want to mess with that.

Finally, let’s run the bootm command in order to trigger the load of the application image from memory…


```console
bootm ${LOADADDR}
```

…and a couple of seconds later we find ourselves in a shell.

![](/assets/posts/tg862/uart_bash.PNG)

This is where things got complicated. Since the init process was not run, the system is not fully initialized. This means that our functionality is quite limited. You might picture the the bootup process as follows (dotted rectangles not run).


![](/assets/posts/tg862/uboot_bash.png)


Even though the init script from ARRIS was not loaded, the box kept resetting after about 40 seconds. According to some log files and scripts that I discovered, there should be a watchdog with a kick interval of 10 seconds. Needless to say that without detailed documentation of the hardware registers of the chip even such a simple task as disabling the watchdog can be challenging.

Currently, my motivation to pursue the problem is limited. However, getting a shell and playing around with the thing was quite fun. Obviously, there is a lot more that can be done reverse engineering the device, especially because we now have full access to all the scripts and binaries of the system.

Here is a short list of what I can think of right away:

- There seems to be a Timer0 loaded during boot (logfiles), however, since I was not able to fetch a datasheet of the CPU don’t know if this timer is used as a watchdog.
- There is still the possibility that JTAG is active and can be used, however, without tools and information about the device this might be quite hard to exploit.
- Use NSA’s Ghidra in order to disassemble some of the binaries to look for backdoors or security vulnerabilities.
- As far as I can tell, the device does not verify the integrity of the image, so it might be possible to upload custom firmware