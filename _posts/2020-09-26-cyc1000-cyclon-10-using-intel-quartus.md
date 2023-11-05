---
title: 'CYC1000 (Cyclon 10) Using Intel Quartus'
permalink: /cyc1000-cyclon-10-using-intel-quartus/
categories:
    - 'FPGA Development'
    - Intel
tags:
    - Arrow
    - CYC1000
    - 'Cyclon 10'
    - Intel
    - 'Quartus Prime'
---

## Intro

At work I have to deal with Xilinx FPGA’s and SoC’s of all kind, however, I have not used any Intel (former Altera) FPGA so far. Since every manufacturer delivers it’s own toolchain I was also not familiar with the Quartus software at all. A couple of weeks ago I was browsing the web and decided that it is time to check out Intel’s product range. I was quite surprised that the whole concept of their website was structured quite similar to the Xilinx webpage. From there on I was hooked, I needed to find out if their toolchain and concept of FPGA development is also quite similar to the Xilinx workflow (or the Intel workflow is similar to the Xilinx approach, whatever perspective you prefer).

## The Hardware

In order to run tests on some actual hardware (that’s always more fun than a simulation) and to make sure that I can experience the whole workflow starting from RTL all the way to bitstream generation and hardware testing, I discovered the CYC1000 board. This small and super simple module seems to be the successor of the MAX1000. It is based on the latest Intel FPGA family, the Cyclone 10 LP and seems to be manufactured by [trenz electronic](https://shop.trenz-electronic.de/en/Products/Trenz-Electronic/CYC1000-Intel-Cyclone-10/), however, the module can currently only be ordered from [Arrow](https://www.arrow.com/en/products/cyc1000/arrow-development-tools). With a price as low as 39$, you will get:

Cyclone 10 25kLP FPGA
- 8 MByte SDRAM
- 2 MByte Flash
- LIS3DH 3-axis acceleration sensor
- LEDs
- Buttons
- PMOD headers
- ARROW USB Blaster

![_config.yml]({{ site.baseurl }}/images/cyclon10/cyc1000.png)


This is how it arrived at my doorstep.

![_config.yml]({{ site.baseurl }}/images/cyclon10/NDP_1949.png)

## The Software

As a first step, download the Intel [Quartus Prime Lite Design Software](https://fpgasoftware.intel.com/?edition=lite). If you want to simulate your circuit, do not forget to also include the ModelSim-Intel FPGA package (the one which includes the Starter Edition) as well as the Cyclone 10 LP device support package.

After the initial setup, let’s start Quartus Prime, open up the “New Project Wizard” and select the correct Cyclone 10 LP (10CL025YU256C8G). Before hitting “Finish”, select “ModelSim-Altera simulator” and your preferred Language.

![_config.yml]({{ site.baseurl }}/images/cyclon10/quartus_cyclon10LP.png)

![_config.yml]({{ site.baseurl }}/images/cyclon10/quartus_modelsim-altera.png)

Although Quartus provides a tool for pin assignment (“Assignment Editor” and “Pin Planner”), I don’t really like the idea to do this within the GUI. First it is slow and second, it is a lot more productive if the assignments are grouped and have proper comments assigned, so that they can be easily reused in another project. Therefore I wrote a TCL script that contains all the pin assignments for the board. In order to make use of them to the current project, first add the TCL file to the project, then select “Tools” → “TCL Scripts” and run the script. Alternatively use the Quartus Prime Tcl Console and execute:

``` bash
source "cyc1000_pins.tcl"
```

In order to verify that everything went fine, make sure that all the pins are assigned successfully in the Quartus “Pin Planner”.

For some additional constraints let’s add a constraints file using “File” → “New” → “Synopsys Design Constraints File”, specify the clock frequency as follows and save the file to your project.

``` code
# input clock definition
create_clock -name clk12m_i -period 83.333 [get_ports {clk12m_i}]
```

***Note**: that since we do not specify any constraints about RX and TX of the UART, Quartus will warn us about this Unconstrained Path in the Timing Analyzer when the project is compiled. Thats not critical for this little test-project however.*

In order to finalize the constraints part of this project, it is a good idea to define the default IO-Standard for the device. Therefore right-click the current project, then select “Settings” → “Device/Board” → “Device and Pin Options” → “3.3-V LVTTL”.

Since the CYC1000 contains an FT2232H which can be used for FPGA configuration using the Arrow-USB-Blaster as well as UART transmission, I thought this would be a nice way to verify both work as expected including the driver. Let’s start right away using “Tools” → “Platform Designer” in order to add an UART RS232 IP in streaming mode (no memory mapping required) with the default settings of 115200 Baud, 1n8.

![_config.yml]({{ site.baseurl }}/images/cyclon10/platform_designer_clock-1.png)

![_config.yml]({{ site.baseurl }}/images/cyclon10/platform_designer_uart.png)

Continue by selecting “clk\_0” with a clock frequency of 12MHz (on-board oscillator) and connected “clk” and “reset”. Finally loopback RX and TX on the streaming side and export the “external\_interface” using the “ext”-prefix as shown below.

![_config.yml]({{ site.baseurl }}/images/cyclon10/platform_designer_connect.png)

As a last step, save the System und generate the output product using “Generate” → “Generate HDL”. In order to add the system to the VHDL top-level design, the easiest method is to copy the Instantiation Template (“Generate” → “Instantiation Template”).

Next create a VHDL top-level file and add it to the project. For the simple loopback test, the following minimalistic code is sufficient.

``` code
library ieee;
use ieee.std_logic_1164.all;
use ieee.numeric_std.all;


entity uart_loopback is

  port (
    clk12m_i   : in std_logic;
    uart_rxd_i : in std_logic;
    uart_txd_o : out std_logic);

end entity uart_loopback;


architecture arch of uart_loopback is

  component loopback is
    port (
      clk_clk       : in  std_logic := 'X';  -- clk
      reset_reset_n : in  std_logic := 'X';  -- reset_n
      ext_RXD       : in  std_logic := 'X';  -- RXD
      ext_TXD       : out std_logic          -- TXD
      );
  end component loopback;

begin

  u0 : component loopback
    port map (
      clk_clk       => clk12m_i,
      reset_reset_n => '1',
      ext_RXD       => uart_rxd_i,
      ext_TXD       => uart_txd_o);


end architecture arch;
```

In order for the design tool to know that this is our top-level, we have to right-click the file in “Project Navigator” and select “Set as Top-Level Entity”. “Start Analysis &amp; Elaboration” and make sure there are no errors before you continue and compile the whole project.

***Note:** I decided not to do any simulation (which you should never even consider doing when writing code), however, this test-project is trivial and does not rely on our code so I skipped that part.*

## The Programmer

Using the programmer is straight forward, but requires a bit of preparation for the first time. Install the libjatag\_hw\_arrow.so in order to allow hardware configuration and debugging. The [Trenz-Wiki](https://wiki.trenz-electronic.de/display/PD/TEI0004+Driver+Setup) provides instructions on how to do this. For whatever reason, the udev rule provided did not work for me, since unbind did not work (running Linux ubuntu 3.19.0-80-generic). I had to change the *RUN* key to *PROGRAM*, although this is not what it is intended for according to the [man page](http://manpages.ubuntu.com/manpages/xenial/man7/udev.7.html). The following udev rule worked for me.

``` code
# Arrow-USB-Programmer
 SUBSYSTEM=="usb",\
 ENV{DEVTYPE}=="usb_device",\
 ATTR{idVendor}=="0403",\
 ATTR{idProduct}=="6010",\
 MODE="0666",\
 NAME="bus/usb/$env{BUSNUM}/$env{DEVNUM}",\
 RUN+="/bin/chmod 0666 %c"
 
# Interface number zero is a JTAG.
 SUBSYSTEM=="usb",\
 ATTR{idVendor}=="0403",\
 ATTR{idProduct}=="6010",\
 ATTR{interface}=="Arrow USB Blaster",\
 ATTR{bInterfaceNumber}=="00",\
 PROGRAM="/bin/sh -c 'echo $kernel > /sys/bus/usb/drivers/ftdi_sio/unbind'"
```

Also do not forget to reload your udev rules when done:

``` bash
sudo udevadm control --reload
```

In order to make sure that everything went fine, verify that the FTDI device is detected when plugged in and there is only one device-file bound to the FTDI UART driver.

``` bash
user@ubuntu:~$ lsusb
Bus 001 Device 005: ID 0403:6010 Future Technology Devices International, Ltd FT2232C Dual USB-UART/FIFO IC
```

``` bash
user@ubuntu:~$ ls -la /dev/ttyUSB* 
crw-rw-r-- 1 root dialout 188, 1 Sep 25 08:41 /dev/ttyUSB1
```

## Run on the Hardware

In order to test the Bitstream’s functionality we have to configure the FPGA. In Quartus Designer, this can be done using the internal programmer (“Tools” → “Programmer”). The PCB also contains a Serial Configuration Flash Memory (QPCQ-A) which allows the FPGA to restore it’s configuration after a power-loss. However, this requires a *Programmer Object File* (.pof). For now, let’s simply configure the FPGA using the generated *SRAM Object File* (.sof) file.

![_config.yml]({{ site.baseurl }}/images/cyclon10/quartus_program_device_arrowblaster.png)

![_config.yml]({{ site.baseurl }}/images/cyclon10/quartus_program_device.png)

Connect to the UART Port using [screen](https://linux.die.net/man/1/screen)or [minicom](https://linux.die.net/man/1/minicom) :

``` bash
user@ubuntu:~$ sudo screen /dev/ttyUSB1 115200 1n8
```

Finally verify that whatever is typed in the serial terminal is looped back to you so that you can see it. This basically verifies that the FTDI performs USB to UART conversion, redirects the transmitted characters to the FPGA which then loops them back to you so you can see them.

![_config.yml]({{ site.baseurl }}/images/cyclon10/uart_loopback.png)

This verifies that both the ARROW-Programmer and the USB to UART port is working as expected. Nice!