---
id: 448
title: 'EBAZ4205 - Again, but this time with Buildroot'
date: '2021-02-18T06:15:28+00:00'
author: admin
layout: post
guid: 'https://embed-me.com/?p=448'
permalink: /ebaz4205-with-buildroot/
wp_featherlight_disable:
    - ''
categories:
    - Uncategorized
---

## Intro

Yes, yes, I know, I have done a couple of [posts](https://embed-me.github.io/ebaz4205-recycle-cheap-crypto-miner-part-1/) on how to get Embedded Linux running on the EBAZ4205 already, but I really like the board and therefore want to write this last post about it. After I received a second EBAZ4205 with limited hardware (no quartz for the PHY mounted) and adapted the current FPGA design to be compatible with both board versions, I decided to give Buildroot a chance this time. Since the setup, build, and tooling is quite easy and straightforward, the post will be even shorter than the previous ones. Let’s get right to it then…

## The Tools

First of all, a short remark. Compared to Yocto, [Buildroot ](https://buildroot.org/)is quite easy to use and the learning curve is not that steep. At least in my experience using a predefined defconfig or even modify an existing one to fit your needs can be done in no-time. The main difference is that Buildroot offers a nice configuration tool that is very similar to the one used by the Linux Kernel and Busybox. Therefore a lot of developers should already be familiar with the workflow. Furthermore, the [documentation](https://buildroot.org/docs.html) is quite good and offers a perfect starting point. However, one of the main drawbacks (at least from my experience) is the very limited dependency system. In short, I find that most of the configuration changes made need a rebuild of the system and this takes quite some time. Nevertheless, Buildroot is a great tool that offers a cross-compilation toolchain, a root filesystem, Linux kernel image, and bootloader for your custom system.

![](https://buildroot.uclibc.org/images/logo.png)

## The Build

Buildroot allows so the use of a “br2-external tree”. This means that you provide your configuration outside the Buildroot tree. There is no need to rebase the GIT Repository to add your changes. This is how I chose to provide the EBAZ4205 configuration. First of all, we need to clone the Buildroot GIT repository.

``` bash
git clone -b 2020.11.x git://git.busybox.net/buildroot
```

Next, the external tree for EBAZ4205 from [my Github Repo](https://github.com/embed-me/ebaz4205_buildroot).  
*Note: This tree, like the Yocto Board Support Package provided in the previous posts, is based on the [embed-me u-boot](https://github.com/embed-me/u-boot) fork for EBAZ4205*.

``` bash
git clone -b 2020.11.x https://github.com/embed-me/ebaz4205_buildroot.git
```

Including the external tree is straightforward. We only need to provide the path to it once during defconfig and from there on Buildroot will know about it.

``` bash
make BR2_EXTERNAL=/path/to/ebaz4205_buildroot zynq_ebaz4205_defconfig
```

Finally we start the build. In order to have detailed logging information about the build, the output is piped into a logfile.

``` bash
make 2>&1 | tee build.log
```

Don’t know what else to say, but that’s basically it. Building the output products can take some time, however, with the current minimal configuration it was significantly faster than building the Yocto Image.

## The Artifacts

All the build artifacts are placed in the subdir *output/images/*. Although a post-build script can be used to generate a bootable image that can be copied to the SD-Card using *dd* (like WIC used by Yocto), I decided not to add support for that. Why? Because with the current flow, the Bitstream is configured using u-boot (not the Xilinx FSBL!). Therefore we need to copy the ebaz4205\_top.bin file from the Vivado Build to the SD-Card anyway. Long story short, the SD-Card need to have at least one FAT partition with the boot flag set and the following files need to be copied on it (correct naming matters):

- boot.bin
- ebaz4205\_top.bin (from the Vivado build)
- ebaz4205-zynq7.dtb
- rootfs.cpio.uboot
- u-boot.bin
- u-boot.img
- uEnv.txt
- uImage

## Verification

To verify that the build was successful and everything is working, let’s try it on the EBAZ4205. A couple of seconds later, we find ourselves in the Buildroot Linux login prompt. Yay!

![_config.yml]({{ site.baseurl }}/images/ebaz4205_part5/prompt.png)

## What’s next

Nothing much to say here, except, see you next time!