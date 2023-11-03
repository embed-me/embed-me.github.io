---
id: 515
title: 'QEMU - How to emulate your Zynq-7000 - Part 2'
date: '2021-05-27T17:52:10+00:00'
author: Lukas Lichtl
layout: post
guid: 'https://embed-me.github.io/?p=515'
permalink: /qemu-how-to-emulate-your-zynq-7000-part-2/
wp_featherlight_disable:
    - ''
categories:
    - Uncategorized
tags:
    - ebaz4205
    - Emulation
    - QEMU
    - Zynq-7000
---

## Intro

In the [previous post](https://embed-me.github.io/qemu-how-to-emulate-your-zynq-7000/) about the Zynq-7000 and QEMU, we took a closer look at emulating a whole Linux using QEMU’s System Emulation feature. This post, will be a little add-on and show how applications compiled for a different architecture can be run on the host through what’s called User Mode Emulation. QEMU has a broad spectrum of features including System Call Translation, POSIX signal handling, and Multi-Threading in order to emulate as much of the User Space as possible. This feature set is quite impressive and its usage is super simple. However, if you are interested in the internal workings of QEMU including a description of how the remarkable dynamic translation works, have a look at [MIT’s documentation of QEMU Internals](https://stuff.mit.edu/afs/sipb/project/phone-project/share/doc/qemu/qemu-tech.html).

![_config.yml]({{ site.baseurl }}/images/zynq_quemu_part2/image.png "QEMU")

## The Prerequisites

The following section assumes that you already have an application compiled for the Cortex-A9 processor including the targets sysroot at hand. If you have not, feel free to build your own using the [instructions of a previous blog post](https://embed-me.github.io/ebaz4205-recycle-cheap-crypto-miner-part-4/). Of cause, you can also compile it using a cross-compiler that your distributions Package Manager offers. Depending on the number of libraries that are used during compilation this can be quite exhaustive, however. Also, ensure that the path to your QEMU binary is set. If you use a manual build, take a look at [part 1](https://embed-me.github.io/qemu-how-to-emulate-your-zynq-7000/) in order to see how this can be done.

Verify successful cross-compilation using the *file* command.

``` bash
$ file helloworld
../test/helloworld: ELF 32-bit LSB shared object, ARM, version 1 (SYSV), dynamically linked (uses shared libs), BuildID[sha1]=59fa1304518f90df65159f0a70525d466b23b168, for GNU/Linux 3.2.0, not stripped
```

Looking at the ELF header, we can verify the above output.

``` bash
$ which $READELF
<path_to_sysroot>/usr/bin/arm-poky-linux-gnueabi/arm-poky-linux-gnueabi-readelf

$ $(READELF) -h helloworld
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Shared object file)
  Machine:                           ARM
  Version:                           0x1
  Entry point address:               0x411
  Start of program headers:          52 (bytes into file)
  Start of section headers:          10092 (bytes into file)
  Flags:                             0x5000400, Version5 EABI, hard-float ABI
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         9
  Size of section headers:           40 (bytes)
  Number of section headers:         37
  Section header string table index: 36
```

## Emulating the Application

Now that we are equipped with a simple demo application, all we have to do is to invoke *qemu-arm* in order to launch a Linux process for us.  
*Note, that we specify the -L option in order to point to our targets sysroot directory.*

``` bash
qemu-arm -cpu cortex-a9 -p 4k -L <path_to_sysroot> helloworld
Hello World from EBAZ4205!
```

That’s basically it – not hard at all.  
*Note, that I also updated the makefile provided in the [part 1](https://embed-me.github.io/qemu-how-to-emulate-your-zynq-7000/), to allow the emulation of user applications in the [ebaz4205\_qemu](https://github.com/embed-me/ebaz4205_qemu) Repository.*

## What’s next

This blog post concludes the short excursion about QEMU. Of cause, this can also come in handy in case you want to debug binaries during reverse engineering – dynamic analysis for example. However, this example was trivially simple, the more dependencies the binary has, the more challenging the emulation will be. Have fun and see you soon!