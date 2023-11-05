---
title: 'QEMU - How to emulate your Zynq-7000'
permalink: /qemu-how-to-emulate-your-zynq-7000/
categories:
    - 'FPGA Development'
    - QEMU
    - SoC
    - Xilinx
tags:
    - ebaz4205
    - Emulation
    - QEMU
    - Zynq-7000
---

## Intro

As you are probably aware, [QEMU ](https://www.qemu.org/)is a quite fast machine emulator and virtualizer. This means that it allows us to emulate hardware with it, and the nice thing is that Xilinx supports it for most of their devices, including the MicroBlaze softcore. Although QEMU supports user-mode emulation (running applications compiled for a different architecture), we will focus on the system-mode for now in order to boot up a Linux system without the actual hardware. Since a lot of work on this blog is already based on the EBAZ4205 board and there are posts available on how to build and tailor a Linux Distro for it, I will use the artifacts of these builds within this blog post. If you want to generate Boot Files for EBAZ4205, instructions are available on the blog posts regarding EBAZ4205 in [Part 2](https://embed-me.github.io/ebaz4205-recycle-cheap-crypto-miner-part-2/) and [Part 3](https://embed-me.github.io/ebaz4205-recycle-cheap-crypto-miner-part-3/) of the series. So let’s get right to it.

![_config.yml]({{ site.baseurl }}/images/zynq_quemu_part1/image.png)

## The Prerequisites

First of all, we need to fetch Xilinx’s fork of QEMU.

``` bash
git clone -b xilinx-v2020.1 https://github.com/Xilinx/qemu.git
```

Depending on your distribution, ensure that the build requirements are fulfilled.  
*On Ubuntu run:*

``` bash
sudo apt-get build-dep qemu
```

Within the QEMU repository, start the build as follows.  
*Note that the build instructions can also be found in the [Xilinx UG1169](https://www.xilinx.com/support/documentation/sw_manuals/xilinx2019_2/ug1169-xilinx-qemu.pdf).*

``` bash
mkdir build
cd build
../configure --target-list="aarch64-softmmu,microblazeel-softmmu,arm-softmmu" --enable-debug --enable-fdt --enable-sdl
make -j8
```

Finally, export the PATH to the QEMU binaries.

``` bash
export PATH=<your-path>/qemu/build/aarch64-softmmu/:$PATH
```

<div aria-hidden="true" class="wp-block-spacer" style="height:10px"></div>## Running QEMU

In order to ease the network configuration on the host, I prepared a simple Makefile. Let’s fetch it from the embed-me [ebaz4205\_qemu](https://github.com/embed-me/ebaz4205_qemu) Repo first.

``` bash
git clone git@github.com:embed-me/ebaz4205_qemu.git
```

Open the Makefile in your favourite Editor and adjust the IMG\_DIR and HOST\_ADAPTER\_TO\_BRIDGE variable as required, then run:

``` bash
make net_prep
```

After running the above commands, the settings of the interfaces and links should look similar to the ones shown below. However, some of the interfaces might still indicate that they are down until the QEMU system is running and connected.

``` bash
$ ip link
1: ens38: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master br0 state UP mode DEFAULT group default qlen 1000
    link/ether de:ad:de:ad:de:ad brd ff:ff:ff:ff:ff:ff
19: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether de:ad:de:ad:de:ad brd ff:ff:ff:ff:ff:ff
20: tap0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master br0 state UP mode DEFAULT group default qlen 1000
    link/ether a2:2e:8a:2d:a5:9d brd ff:ff:ff:ff:ff:ff
```

Then finally run QEMU.  
*(You will need the WIC image in order to emulate the rootfs for your device.)*

``` bash
make run_system
```

And after the boot, we should be greeted with the login prompt.

``` bash
...
[  OK  ] Started RPC Bind Service.
[  OK  ] Started NFS status monitor for NFSv2/3 locking..
[  OK  ] Started Login Service.
[  OK  ] Reached target Multi-User System.
         Starting Update UTMP about System Runlevel Changes...
[  OK  ] Started Update UTMP about System Runlevel Changes.

EBAZ4205 Distro 1.0 ebaz4205-zynq7 ttyPS0

ebaz4205-zynq7 login:
```

The rootfs and our boot partition are mounted.

``` bash
root@ebaz4205-zynq7:~# ls -ladia/mmcblk0p1/
drwxr-xr-x    2 root     root         16384 Jan  1  1970 .
drwxr-xr-x    3 root     root             0 Feb 16 20:06 ..
-rwxr-xr-x    1 root     root       2764976 Mar 10 15:47 boot.bin
-rwxr-xr-x    1 root     root      19260990 Mar 10 15:47 ebaz4205-image-standard-ebaz4205-zynq7.cpio.gz.u-boot
-rwxr-xr-x    1 root     root         22648 Mar 10 15:47 ebaz4205-zynq7.dtb
-rwxr-xr-x    1 root     root        576788 Mar 10 15:47 u-boot.img
-rwxr-xr-x    1 root     root           538 Mar 10 15:47 uEnv.txt
-rwxr-xr-x    1 root     root       4302072 Mar 10 15:47 uImage
-rwxr-xr-x    1 root     root       4302008 Mar 10 15:47 zImage
```

There should also be an IP address assigned by the DHCP server.

``` bash
root@ebaz4205-zynq7:~# ip addr     
...
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000
    link/ether 02:f0:0d:ba:be:02 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.92/24 brd 10.0.0.255 scope global dynamic eth0
       valid_lft 863959sec preferred_lft 863959sec
    inet6 fe80::f0:dff:feba:be02/64 scope link
       valid_lft forever preferred_lft forever
...
```

Back at the host, you are now able to log into your system using SSH as you would in the actual device.

``` bash
$ ssh root@10.0.0.92
root@10.0.0.92's password:
root@ebaz4205-zynq7:~#
```

In order to exit QEMU press *“Ctrl-a x”*. Switching between the console of QEMU und the emulated system can be done with *“Ctrl-a c”*!

Once you are done, cleanup the network interfaces (tap and bridge) that were initially generated using:

``` bash
make net_unprep
```

Up until now, you are able to emulate the system and test new build artifacts of your Embedded Linux Toolchain without the actual hardware. This is already quite nice and can speed up development workflow significantly. However, QEMU offers much more features including great debugging support – even for the kernel ([using vmlinux and GDB](https://qemu-project.gitlab.io/qemu/system/gdb.html)). The opportunities are quite endless and I hope this post offers a good starting point for you. If you are more interested in User Mode Emulation, make sure to check out [part 2](https://embed-me.github.io/qemu-how-to-emulate-your-zynq-7000-part-2/). See you next time 😉