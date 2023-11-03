---
id: 533
title: 'EBAZ4205 - A cheap network-based JTAG debugger'
date: '2021-06-12T06:43:48+00:00'
author: admin
layout: post
guid: 'https://embed-me.com/?p=533'
permalink: /ebaz4205-a-cheap-network-jtag-debugger-using-xvc/
wp_featherlight_disable:
    - ''
categories:
    - Uncategorized
---

## Intro

I always wanted to have a cheap debugger tightly integrated into the Vivado Toolchain. So why not use an EBAZ4205 to debug another EBAZ4205? I thought that this might be a fun thing to do and reminded me of the way one can use a cheap STM32 to program another STM32 compatible board (eg. to get the excellent functionality of the [Black Magic debugging tool](https://github.com/blacksphere/blackmagic)). It turns out that all relevant logic levels of the board are 3V3 and the JTAG signals, in general, are typically quite slow, so I do not expect any hardware issues. Let’s give it a try then 😉

## The Setup

In order to clarify, the following figure shows how I expect the setup to look like. The target is connected to the debugger through JTAG, which in turn is connected to the Host computer with Ethernet running [XVC](https://www.xilinx.com/products/intellectual-property/xvc.html) (Xilinx Virtual Cable) protocol on top of TCP/IP.

![_config.yml]({{ site.baseurl }}/images/ebaz4205_network_jtag/xvc-server.png)

The following image shows the above scenario with actual hardware. The wires connecting both boards are the JTAG ports.

![_config.yml]({{ site.baseurl }}/images/ebaz4205_network_jtag/ebaz4205-jtag_setup.png)

## The FPGA Design

First, let’s create a Vivado Design which includes a so-called [Xilinx Debug Bridge](https://www.xilinx.com/products/intellectual-property/debug-bridge.html). It supports multiple modes which can be used to assist you in debugging your design, however, it also allows you to interface with an FPGA through JTAG. Luckily exactly what we want and trivial to set up. The following figure shows the resulting Block Design containing PS (Processing System), Interconnect, System Reset, and Debug Bridge. If you are new to the EBAZ4205 hardware platform, make sure to check out [this post](https://embed-me.github.io/ebaz4205-recycle-cheap-crypto-miner-part-2/) before you continue.

![_config.yml]({{ site.baseurl }}/images/ebaz4205_network_jtag/ebaz4205_jtag_bd.png)

All that’s left is to define the pinout in a [constraints file](https://www.xilinx.com/support/documentation/sw_manuals/xilinx2019_2/ug945-vivado-using-constraints-tutorial.pdf). The following pins are all located on the “DATA1” connector on the PCB and therefore easily accessible with wires.

| JTAG-Function | FPGA-Signal | EBAZ4205-CONNECTOR | FPGA-Pin |
|---|---|---|---|
| TCK | tck\_o | DATA1-15 | H18 |
| TMS | tms\_o | DATA1-16 | D19 |
| TDO | tdo\_o | DATA1-20 | K17 |
| TDI | tdi\_o | DATA1-19 | F19 |

## The Software

Up until now, everything was managed in the FPGA. It’s time to integrate it into the Yocto Image (have a look at this [previous post](https://embed-me.github.io/ebaz4205-recycle-cheap-crypto-miner-part-3/) in order to get started). The first thing to do is to add a node in the “amba” device-tree section which contains the full details of the Debug Bridge.   
*Note, that the address might be different depending on how it was memory-mapped in the previous step.* *I have not checked the Linux driver, however, most of the parameters are probably not even used and could therefore be stripped.*

``` c
debug_bridge_0: debug_bridge@43c00000 {
	clock-names = "s_axi_aclk";
	clocks = <&clkc 15>;
	compatible = "xlnx,debug-bridge-3.0", "generic-uio";
	reg = <0x43c00000 0x10000>;
	xlnx,bscan-mux = <0x1>;
	xlnx,build-revision = <0x0>;
	xlnx,chip-id = <0x0>;
	xlnx,clk-input-freq-hz = <0x17d7840>;
	xlnx,core-major-ver = <0x1>;
	xlnx,core-minor-alpha-ver = <0x61>;
	xlnx,core-minor-ver = <0x0>;
	xlnx,core-type = <0x1>;
	xlnx,dclk-has-reset = <0x0>;
	xlnx,debug-mode = <0x3>;
	xlnx,design-type = <0x0>;
	xlnx,device-family = <0x0>;
	xlnx,en-bscanid-vec = "false";
	xlnx,en-int-sim = <0x1>;
	xlnx,en-passthrough = <0x0>;
	xlnx,enable-clk-divider = "false";
	xlnx,fifo-style = "SUBCORE";
	xlnx,ir-id-instr = <0x0>;
	xlnx,ir-user1-instr = <0x0>;
	xlnx,ir-width = <0x0>;
	xlnx,major-version = <0xe>;
	xlnx,master-intf-type = <0x1>;
	xlnx,minor-version = <0x1>;
	xlnx,num-bs-master = <0x0>;
	xlnx,pcie-ext-cfg-base-addr = <0x400>;
	xlnx,pcie-ext-cfg-next-ptr = <0x000>;
	xlnx,pcie-ext-cfg-vsec-id = <0x0008>;
	xlnx,pcie-ext-cfg-vsec-length = <0x020>;
	xlnx,pcie-ext-cfg-vsec-rev-id = <0x0>;
	xlnx,tck-clock-ratio = <0x8>;
	xlnx,two-prim-mode = "false";
	xlnx,use-bufr = <0x0>;
	xlnx,use-ext-bscan = "true";
	xlnx,use-softbscan = <0x1>;
	xlnx,use-startup-clk = "false";
	xlnx,user-scan-chain = <0x1>;
	xlnx,xsdb-num-slaves = <0x0>;
	xlnx,xvc-hw-id = <0x0001>;
	xlnx,xvc-sw-id = <0x0001>;
};
```

Next, let’s add the XVC-server to the *ebaz4205-image-standard* recipe so that the application will be included. The recipe compiles the Xilinx XVC-server source code and a systemd service to start it automatically once a network connection was established.

``` bash
IMAGE_INSTALL += "xvc-server "
```

In order for the previously generated custom XSA to be used, the *external-hdf.bbappend* needs to be adjusted, so that  
it will be fetched locally instead of from the server (in case you copy it into the files directory, it might look like below).

``` bash
FILESEXTRAPATHS_prepend := "${THISDIR}/files:"

HDF_BASE = "file://"

HDF_PATH = "ebaz4205_top_xvc.xsa"
HDF_NAME = "ebaz4205_top_xvc.xsa"

HDF_MACHINE = ""
S = "${WORKDIR}"
```

Now, let’s build the image using Bitbake.

``` bash
bitbake ebaz4205-image-standard-wic
```

And finally write it to the SD-Card.

``` bash
$ dd if=tmp/deploy/images/ebaz4205-zynq7/ebaz4205-image-standard-wic-ebaz4205-zynq7.wic of=/dev/<your_dev> bs=4096
36752+1 records in
36752+1 records out
150537216 bytes (151 MB) copied, 33.5964 s, 4.5 MB/s
```

## Using the debugger

Once the EBAZ4205 acting as a JTAG bridge is powered up, we need to open the Hardware Manager and connect to the XVC-Server running on port 2542. That’s actually quite simple to do from within Vivado’s TCL shell:

``` tcl
open_hw_manager
connect_hw_server
open_hw_target -xvc_url <ebaz4205_jtag_ip>:2542
```

We can already see and fully control our target. The following figures show the target before- and after programming the FPGA.

![_config.yml]({{ site.baseurl }}/images/ebaz4205_network_jtag/xvc-not-programmed.png)

![_config.yml]({{ site.baseurl }}/images/ebaz4205_network_jtag/xvc-programmed.png)

## What’s next

In case you want to rebuild the system or fiddle around with it yourself, the sources for the FPGA design are available on the embed-me [ebaz4205\_fpga ](https://github.com/embed-me/ebaz4205_fpga/tree/rel-v2020.1-xvc)Github repo! The basic ebaz4205 Yocto BSP that was modified can also be found on the [embed-me Github repo](https://github.com/embed-me).  
For me, this was a really fun feature to implement and I hope you enjoyed reading it 🙂