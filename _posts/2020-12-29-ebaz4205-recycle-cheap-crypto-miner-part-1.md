---
title: 'EBAZ4205 - "Recycle" a cheap crypto-miner (Part 1)'
permalink: /ebaz4205-recycle-cheap-crypto-miner-part-1/
categories:
    - 'FPGA Development'
    - SoC
    - Xilinx
tags:
    - Bring-up
    - FPGA
    - PCB
    - SoC
    - z7010
---

## Intro

A couple of weeks ago I read an interesting post on Reddit ([here](https://www.reddit.com/r/FPGA/comments/jvjskn/anyone_have_experience_with_chinese_zynq_boards/), and also [here](https://www.reddit.com/r/hackaday/comments/jwf3h4/hacking_the_fpga_control_board_from_a_bitcoin/)). Seems like some crypto-mining farms were closed or reworked and therefore the online market was flooded with what seems to be crypto-miner control boards. The interesting stuff is that these are based on a Zynq z7010 and are sold for as low as 10€ per piece on [Aliexpress](https://www.aliexpress.com/wholesale?SearchText=zynq+7000). If you compare these PCBs with the price of the Zynq alone, this is really really cheap! Although there are already multiple [posts](https://github.com/xjtuecho/EBAZ4205) mentioning the schematic and hardware in use, most of them lack details on the FPGA and Linux point of view. In the upcoming series of posts, I will try to tests its functionality, document problems or pitfalls (if any) encountered during the hardware “bring-up”, and provide some form of BSP (Board Support Package) for the board so that it can be used in any project.

## The Hardware

No matter where you order, usually have two options, the board as is (with boot device NAND Flash) or with modifications so that the boot mode is set to SD-Card. I ordered the version with SD-Card, UART- and JTAG-Header already mounted. The condition of the board was quite nice, with only some small scratches on the bottom. After I cleaned the soldering pads, the PCB was almost perfect.

![](/assets/posts/ebaz4205_part1/ebaz4205_bot_2.png)

Some of the main components on the board, including headers, are highlighted below. The best resource for the schematic is probably [here](https://github.com/xjtuecho/EBAZ4205). However, be aware, that the schematic does not reflect the state of the board to 100% when ordered with “SD-Card mounted”, since the boot mode pins were altered!

![](/assets/posts/ebaz4205_part1/nbaz4205_top_components.png)

**Explanation:**

- Zynq XC7Z010-1CLG400C (green)
- Ethernet PHY 10BASE-T/100BASE-TX (orange)
- EtronTech 256MB DDR3 (yellow)
- Winbond NAND Flash 128MB (blue)
- 33.333MHz Oszillator for PS (orange)
- UART-Header J7 (white)
- JTAG-Header J8 (magenta)
- …

## The Setup

With the version of the PCB that I received, it was not possible to power the board with J4 (power connector at the bottom right in the image above). The reason for this is, that D24 is marked on the schematic as NC (not connected), and indeed, the diode is missing on the PCB. Therefore I had to use Pin 1/3 on any of the connectors DATA1-3. The board requires 12V on these pins and uses about 60mA if no SD-Card is inserted (in case you want to make sure it does not light up when connected for the first time). A simple USB to UART module connected to the pins on J7 (see above) will provide us a debugging interface.

![](/assets/posts/ebaz4205_part1/ebaz4205_connected.png)

## The Software

I used the following [Buildroot image](https://drive.google.com/file/d/16wQKpiYsH0gQ7KmnnYMrmFdCxwkF3WS4/view?usp=sharing), which was already verified to work in order to boot the board for the first time. I also created a [mirror](https://mega.nz/file/sN4kgQCS#7P3qanUhiiZYH8z0iqO4ExUfTPlSWlgE7A2JuavI8C4) in case the link does not work any longer (MD5 of the image: e5349351fc30bfcf91e4fa5b8e3b3cbe). If you are on Windows, Win32 Disk Imager can be used to write the image to an SD-Card. On Linux, using dd is probably the simplest method.

![](/assets/posts/ebaz4205_part1/win32_disk_imager.png)

Once the system is up and running you are greeted with a serial terminal. For this image the login is *root/root*.

![](/assets/posts/ebaz4205_part1/ebaz4205_login_preb.png)

At this point, we have come quite far with minimal work. We already know that the Power Supply, Zynq, DDR, and hardware modifications for SD-Boot are fine! I decided that the PHY/Ethernet is probably something that also needs to be checked at this point. Therefore, I modified the existing network configuration in */etc/network/interface*s to make use of DHCP:

``` bash
auto eth0
iface eth0 inet dhcp
```

Finally, I made sure that the Port on my local gateway supports 100MBit (correlates with the PHYs capabilities) and also that DHCP is enabled. The figure below shows the end of the boot process when *[udhcpc ](https://udhcp.busybox.net/README.udhcpc)*gets an IP address assigned and we are able to send some greetings to google.

![](/assets/posts/ebaz4205_part1/ebaz4205_phy_preb.png)

I know, there is still some part missing in our verification – the NAND Flash, however, since there was no device in */sys/class/mtd/*, I assume that the flash was not part of the device-tree and/or the driver is not compiled into the kernel. TBH, I don’t really care about this right now and will deal with it at a later point.

## What’s next

I think we came quite far with only a couple of minutes of work! Of cause, this post relied on work that has previously been done by others, but it acts as a starting point. We did not write a single line of code, nor we had to deal with Yocto, Buildroot, or a bare-metal application for verification, and still, we can be quite certain that our hardware works.

In [Part 2](https://embed-me.github.io/ebaz4205-recycle-cheap-crypto-miner-part-2/) of this series, we will make our own FPGA design using Vivado. This will then act as a starting point for our custom Linux distribution for the EBAZ4205 board.