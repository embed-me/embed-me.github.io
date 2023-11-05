---
title: 'EBAZ4205 – “Recycle” a cheap crypto-miner (Part 2)'
permalink: /ebaz4205-recycle-cheap-crypto-miner-part-2/
categories:
    - 'FPGA Development'
    - SoC
    - Xilinx
tags:
    - BSP
    - ebaz4205
    - SoC
    - Vivado
    - xilinx
---

## Intro

I have been working in industry for years now and every company seems to have the same problem when developing with SoC’s and FPGA’s. You need to have a good design flow, some kind of build script or automation in place in order to handle your daily business, use some sort of version control, make optimal use of your resources and do not waste your time on building your project(s) all over again. Automation is key. That’s why I decided to focus not only on the FPGA design itself, but also provide an infrastructure for you to build your project with minimal effort. The perfect design for this is minimalistic – we do not want to write any project specific code, the most basic design that is required to support the EB4205 hardware is a perfect starting point. Let’s dive right into it…

## The Hardware

As already described in [Part 1](https://embed-me.github.io/ebaz4205-recycle-cheap-crypto-miner-part-1/) of the series, J8 is the JTAG header. Since we want to test the design (Bitstream) once we are done, we need to connect to it with our Programmer. Typically Digilent provides a quite cheap alternative to the Xilinx Programmer. I can definitely recommend it! Furthermore, we need to disconnect the SD-Card from the slot, since the default boot mode is set to SD-Card, the board would load the BootROM, FSBL, and U-Boot which will also configure the FPGA with the corresponding bitstream. Finally, we do not really need UART, so you can disconnect it as well if you want to. This is how my setup looks like.

![_config.yml]({{ site.baseurl }}/images/ebaz4205_part2/ebaz4205_top_jtag.png)

## The Build

This section will describe the FPGA build process. As a prerequisite we need to have Vivado Design Suite 2020.1 installed. Feel free to download it at the [Xilinx Website](https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/vivado-design-tools/2020-1.html). Furthermore, I decided to use [hdlmake](https://hdlmake.readthedocs.io/en/master/) for this project, so make sure that you have it available too.

First let’s clone my github [ebaz4205\_fpga repo](https://github.com/embed-me/ebaz4205_fpga), more specific, for this post I suggest you to use the rel-v2020.1 branch.

``` bash
git clone -b rel-v2020.1 https://github.com/embed-me/ebaz4205_fpga.git
```

Navigate to the build directory of the repo and run hdlmake on it. This will generate a Makefile that handles the whole FPGA build.

``` bash
cd build
hdlmake
```

In order to trigger the build, all you have to do is run make, still from within the build directory. Since the design is very small, the build time should be rather fast.

``` bash
make
```

You should see “*touch bitstream*” on the screen when done and the terminal returns. That’s it, the build is done and the bitstream ready! In the current directory you should also see a ebaz4205\_top.xsa, but we will deal with the hardware handoff in the next part of the series. For now, let’s try to open up the design in the GUI. Still within the build directory, the following command should open up the design for us:

``` bash
vivado ebaz4205_top.xpr
```

The following figure shows Vivado, with the block design opened.

![_config.yml]({{ site.baseurl }}/images/ebaz4205_part2/ebaz4205_fpga_design.png)

I already configured the PS (Processing System) side of the chip for the hardware (DDR, …). The following figure describes the setup pretty well – note the checkmarks for enabled modules. I won’t go into detail, basically I read the schematic and modified the MIO’s as required by the hardware.

![_config.yml]({{ site.baseurl }}/images/ebaz4205_part2/ebaz4205_fpga_design2.png)

Once you open the implemented design, it becomes quite clear, that there is almost nothing instantiated on the PL (Programmable Logic, FPGA) part of the chip. That’s your future playground 😉.

![_config.yml]({{ site.baseurl }}/images/ebaz4205_part2/ebaz4205_fpga_utilization1.png)

## The Configuration

Next, let’s open up the Hardware Manager and try to connect to the SoC using JTAG.

![_config.yml]({{ site.baseurl }}/images/ebaz4205_part2/ebaz4205_fpga_hardware_manager2.png)

Now that we are connected to the target, we can observe two things, first, the chip in currently not programmed – that makes sense, we removed the SD-Card before power-up and the chip is set up to boot from there, and second “config done” LED (LED1 on the PCB) is off. Let’s change this and configure the FPGA with the Bitstream that we previously built:

![_config.yml]({{ site.baseurl }}/images/ebaz4205_part2/ebaz4205_fpga_configure.png)

If everything went well, the state of the chip should change to “Programmed” and LED1 should turn ON.

Congratulations, you just programmed the FPGA over JTAG for the first time!

## What’s next

Now, that we are also sure that our FPGA design can be configured, we are also confident that the JTAG interface works fine. This is especially useful if we deal with bare-metal applications, require only the FPGA part of the SoC or need to do some debugging. Sure this was not a guide on how to write VHDL, handle the Vivado toolchain or setup a project, because I think there are already plenty of good resources out there, but what you get is a design tailored to the EBAZ4205. Feel free to alter the design for your project and start building automatically!

In the next part ([Part 3](https://embed-me.github.io/ebaz4205-recycle-cheap-crypto-miner-part-3/)) of the series, I will provide a meta-ebaz4205 layer that you can use in combination with Yocto in order to build your own Linux distribution. The current design builds the foundation. I think that’s quite neat and I am looking forward to it.

See you soon!

**Note: There seem to be two hardware versions currently sold, one with a PHY oscillator mounted and one where it is missing. The design in the Github Repo supports both versions!**