---
title: 'EBAZ4205 – “Recycle” a cheap crypto-miner (Part 3)'
permalink: /ebaz4205-recycle-cheap-crypto-miner-part-3/
categories:
    - 'FPGA Development'
    - SoC
    - Xilinx
    - Yocto
tags:
    - BSP
    - ebaz4205
    - SoC
    - Vivado
    - yocto
    - zeus
---

## Intro

The capabilities of today’s SoC’s, especially in combination with FPGA’s are quite fascinating. You really can do anything with them, and of course, you can run an Embedded Linux! The process of building an Embedded Linux is quite the opposite of HDL development discussed in [Part 2](https://embed-me.com/ebaz4205-recycle-cheap-crypto-miner-part-2/). You are high-level and cannot deal with all the details of the system anymore, there is just too much going on under the hood. Before I started my career as an FPGA engineer I had almost no knowledge of Linux (shame on me), however, since then I learned quite a bit and even prefer the terminal over the GUI for the most part. If you are coming from a Windows background, however, the learning curve can be quite steep. Today we will build our own Linux distribution that is tailored to the EBAZ4205 PCB. As with the posts before, I will try to get you started as quickly as possible. Let’s go!

## The Tools

If you want to create an Embedded Linux for your device, you basically have two options: [Yocto ](https://www.yoctoproject.org/)and [Buildroot](https://buildroot.org/). Buildroot is considered the entry-level, it makes extended use of the Makefile language to configure your build. Yocto uses Bitbake as a build engine and allows more freedom in your configuration, but this obviously results in higher complexity. The huge advantage of Yocto compared to Bitbake is dependency management – in other words, you do not need to rebuild the whole system every time – Yocto keeps track of it. In the end, it takes quite some time to learn the basics, and mastering Yocto is a challenge in itself. Also, you might have heard about Petalinux, which is Xilinx toolchain for building and deploying an Embedded Linux for their chips. However, Petalinux is also based on Yocto!

![](https://docs.yoctoproject.org/_static/YoctoProject_Logo_RGB.jpg)

## The Build

In order to get you started quickly, I already prepared a meta-layer for the EBAZ4205, which we will use as a starting point. Make sure that you have enough space on your disk *since the build directory will become huge.*

First, we need to prepare the build environment. Yocto provides what is called an [Extensible SDK](https://www.yoctoproject.org/docs/3.0.3/mega-manual/mega-manual.html#sdk-installing-the-extensible-sdk). This is basically a tarball installer that includes the pre-built toolchain. For the Yocto “zeus” release we need to download and install (typically to /opt/poky/&lt;version&gt;) the following build tool version:

``` bash
wget http://downloads.yoctoproject.org/releases/yocto/yocto-3.0.4/buildtools/x86_64-buildtools-nativesdk-standalone-3.0.4.sh
./x86_64-buildtools-nativesdk-standalone-3.0.4.sh
```

<div aria-hidden="true" class="wp-block-spacer" style="height:12px"></div>![](/assets/posts/ebaz4205_part3/ebaz4205_install_esdk.png)

Next, we need to clone the poky repository from the Yocto Project.

``` bash
git clone -b zeus git://git.yoctoproject.org/poky
cd poky/
```

Within the repository, we have to add all the meta-layers that we need. In our case, we have to fetch Xilinx, Openembedded, and the custom layer for the EBAZ4205. Let’s clone all of those.

``` bash
git clone -b zeus --depth=1 https://github.com/Xilinx/meta-xilinx.git
git clone -b zeus --depth=1 https://github.com/openembedded/meta-openembedded.git
git clone -b rel-v2020.1 --depth=1 https://github.com/Xilinx/meta-xilinx-tools.git
git clone -b zeus --depth=1 https://github.com/embed-me/meta-ebaz4205.git
```

<div aria-hidden="true" class="wp-block-spacer" style="height:12px"></div>![](/assets/posts/ebaz4205_part3/ebaz4205_poky_dir.png)

In order to allow easy setup, I have created a [template configuration](https://www.yoctoproject.org/docs/3.0.3/mega-manual/mega-manual.html#creating-a-custom-template-configuration-directory) file. To ensure it’s usage, we have to copy it to the poky root.

``` bash
cp meta-ebaz4205/conf/.templateconf .
```

Finally, we prepare our build environment as follows.   
(*Note, that the path to the first environment-setup script might be different if you changed the default installation path.*)

``` bash
source /opt/poky/3.0.4/environment-setup-x86_64-pokysdk-linux
source oe-init-build-env
```

In order to trigger the build of an image (all the files required to boot the device), let’s start Bitbake as follows:

``` batch
bitbake ebaz4205-image-standard-wic
```

If you do not have a build-server, “Threadripper” or similar CPU, the build can take a couple of hours, so you better grab a cup of coffee, or two 😉

## The Artefacts

Once the build is completed, we need to locate the build artifacts and store them on an SD-Card. With the WIC image that we built, this is quite simple.  
*Note: Make 100% sure that &lt;dev&gt; is really your SD-Card and double-check twice!*

``` bash
cd tmp/deploy/images/ebaz4205-zynq7
dd if=ebaz4205-image-standard-wic-ebaz4205-zynq7.wic of=/dev/<dev> bs=4096
```

You will end up with two partitions, one with a FAT16 (Partition 1, boot) File System and another one with EXT4 (Partition 2, root). Currently, only the boot partition (1) is used, the second partition might be used in the future to implement an [overlay](https://www.kernel.org/doc/html/latest/filesystems/overlayfs.html?highlight=overlayfs).

## Verification

Stick the card into the board, connect your USB-UART Cable with a Baud rate of 115200, 1 stop bit, *and no parity*. If you use [screen](https://www.man7.org/linux/man-pages/man1/screen.1.html), the following command should be fine.

``` bash
screen /dev/ttyUSB0 115200 1n8
```

Now power the board up and you should see it boot right away.

![](/assets/posts/ebaz4205_part3/ebaz4205_linux_login.png)

The default login on the command prompt is *root/root*. For detailed information please have a look at the [embed-me/meta-ebaz4205](https://github.com/embed-me/meta-ebaz4205) Github repo’s README file.

## What’s next

It is up to you now to tailor the system to your requirements. The good news is, that people already have done plenty of work that makes your lives easier. You do not need to write your own recipe for the most common tools, simply browse [openembedded layers](https://layers.openembedded.org/layerindex/branch/master/recipes/), add the recipe to the ebaz4205 image and rebuild 😉 If you want to go into detail I can really recommend the book [Embedded Linux Systems with the Yocto Project](https://www.goodreads.com/book/show/18965166-embedded-linux-systems-with-the-yocto-project?from_search=true&from_srp=true&qid=0vWIOJXP07&rank=2) and after that of cause the [Yocto Mega-Manual](https://www.yoctoproject.org/docs/3.1.2/mega-manual/mega-manual.html). Also do not get frustrated, it takes quite some time to make sense of the build environment and its capabilities. Finally, since you really liked this series of posts so much, I decided to do another one on how to cross-compile applications for the Zynq’s ARM Architecture. So let’s keep digging in [Part 4](https://embed-me.github.io/ebaz4205-recycle-cheap-crypto-miner-part-4/)!