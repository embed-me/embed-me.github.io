---
id: 465
title: 'From HDL to Linux Userland'
date: '2021-03-07T20:17:52+00:00'
author: Lukas Lichtl
layout: post
guid: 'https://embed-me.github.io/?p=465'
permalink: /from-hdl-to-linux-userland/
wp_featherlight_disable:
    - ''
categories:
    - Uncategorized
---

## Intro

There are a lot of tutorials and explanations about how to create design, build and integrate IPs into the [Vivado Design Suite](https://www.xilinx.com/products/design-tools/vivado.html) Block Design online, however, when it comes to full-featured support all the way from PL up to Linux, the air gets thinner. I assume that is due to the fact, that not a lot of people do have expertise in all that subjects. I mean, typically there are experts for the FPGA (eg. Digital Design Engineers), then there are the Guys managing the integration into the OS using Yocto or Buildroot for example. Finally, there is yet another Engineer developing the Linux Kernel Driver. In cooperations, this approach makes sense, since you can become an expert in your area and handle the task faster and with fewer errors, on the other hand, you completely lack the broader picture when your responsibility ends. Taking a step back – this is what this blog post is all about!

## The Problem

During a project that I was recently involved in, I noticed that there was no elegant solution to perform PWM ([Pulse Width Modulation](https://en.wikipedia.org/wiki/Pulse-width_modulation)) using standard Xilinx IPs. The only way to archive this was to instantiate a full-blown AXI Timer and use both 32bit counters to generate a PWM. However, even then there was no official Linux Device Driver available. Therefore, in the upcoming sections, I will describe how I designed one on my own. Typically in such a case, it makes the most sense to me to develop bottom-up. Implement the IP first, then verify its functionality and finally move one layer of abstraction up. Let’s start with the FPGA.

## The FPGA

The PWM is done in the FPGA – called PL (=Programmable Logic) in the Zynq – obviously. First, let’s define the Entity (= Interface) of our module with a couple of basic functionalities in mind:

- Enable/Disable the Core
- Invert the PWM signal
- Provide a Period and Duty Cycle
- Trigger a load of the new counter values

``` c
entity pwm_core_rtl is
  generic (
    CNT_WIDTH_G : integer := 32);
  port (
    clk_i			: in  std_logic;
    en_i			: in  std_logic;
    rstn_i			: in  std_logic;
    load_i			: in  std_logic;
    invert_i		: in  std_logic;
    period_cnt_i	: in  std_logic_vector(CNT_WIDTH_G-1 downto 0);
    dutycycle_cnt_i	: in  std_logic_vector(CNT_WIDTH_G-1 downto 0);
    pwm_o			: out std_logic);
```

I won’t go into the details of the implementation, however, the core basically only consists of a couple of registers and some logic. Finally, we wrap this code into an AXI Register Interface to allow the processor to Memory Map the IP and access the registers.

### Code Verification

Once we are confident that the code can be synthesized we can move to verification. Even though we do not design an ASIC here, verification is a crucial step in developing! Even for such a simple IP, defining multiple test cases to cover the normal use case is a must. For this core, I used [VUnit](https://vunit.github.io/) in combination with the [ModelSim](https://eda.sw.siemens.com/en-US/ic/modelsim/) Simulator and defined the following test cases incl. criteria for pass/fail.

- Check that we can read registers
- Set the PWM to a very short Period with 50% Duty Cycle
- Set the PWM to a long Period with 50% Duty Cycle
- Disable the PWM after it was enabled
- Hot-Reload the Period and Duty Cycle Registers while the PWM is enabled
- Set the PWM to generate an uneven Duty Cycle (eg. 1%)
- Invert an uneven Duty Cycle
- Set a 100% Duty Cycle
- Set a 0% Duty Cycle

All tests have a proper check/verification procedure to ensure that they are correct. The following figure shows the summary of VUnit after all tests passed.

![_config.yml]({{ site.baseurl }}/images/hdl_to_linux_userland/vunit_tests_passed.png)

Note that I used VUnits [Verification Component Library](https://vunit.github.io/verification_components/user_guide.html) in order to interface the custom IP using an AXI-Lite interface.   
Finally, we pack the IP in order to be able to insert it into the Xilinx BD without issues. This is the final product of the FPGA development:

![_config.yml]({{ site.baseurl }}/images/hdl_to_linux_userland/pwm_ip_core.png)

Within the main design instantiate, connect, and Memory-Map the IP into the address space of the processor. This is done using the Vivado Address Editor and might look as follows.

![_config.yml]({{ site.baseurl }}/images/hdl_to_linux_userland/pwm_ip_mm.png)

## The Operating System (Linux)

Today’s Embedded FPGA SoC’s are quite complex and most projects require you to install some sort of Operating System. This allows you to benefit from open source applications and it also acts as a layer of abstraction.

### Device-Tree

How does the Linux Kernel even know about our non-hot-pluggable device? There are multiple methods, but the one used the most is using a so-called device-tree. I won’t go into the details of how device-nodes (sections of the device-tree that represent devices) have to be structured, but the main interface to the IP Core is obviously a base register offset and the size of the memory-mapped area. Furthermore, we have to link our device to a driver using the compatibility property. If you want to become a device-tree master, have a look at the specification hosted at [devicetree.org](https://www.devicetree.org/). In our simple case we end up with the following node:

``` c
pwm0: pwm@43C00000 {
		#pwm-cells = <2>;
		clocks = <&clkc 15>;
		clock-names = "fclk0";
		compatible = "embedme,embedme-pwm";
		reg = <0x43C00000 0x10000>;
};
```

In order to follow the UNIX-like mentality, “everything is a file”, we need to provide the user with a standard interface through device-files / the sys-fs. When these files are opened, read, or written to using the standard interface of the Kernel ([System calls](https://en.wikipedia.org/wiki/System_call)), the corresponding callback function in the driver is executed.

In this example, the PWM driver will be a platform-driver, which is a special kind of driver. In order to link the device specified in the device-tree with the driver, we have to provide a struct that contains the same compatibility string used in the device-tree.

``` c
static const struct of_device_id ebme_pwm_dt_ids[] = {
	{ .compatible = "ebme,ebme-pwm", },
	{ /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, ebme_pwm_dt_ids);
```

Next, we construct the corresponding file operation callback functions, that I mentioned earlier.

``` c
static const struct pwm_ops ebme_pwm_ops = {
	.config = ebme_pwm_config,
	.set_polarity = ebme_pwm_polarity,
	.enable = ebme_pwm_enable,
	.disable = ebme_pwm_disable,
	.owner = THIS_MODULE,
};
```

Enabling the PWM for example then becomes a mather of writing the Enable Bit in the Control Register of the IP-Core in the PL.

``` c
static int ebme_pwm_enable(struct pwm_chip *chip, struct pwm_device *pwm)
{
	[...]

	reg = readl(pwm_core->base + REG_CTRL);        
	writel(reg | (CTRL_ENABLE), pwm_core->base + REG_CTRL);

	[...]
}
```

I decided to build a kernel module. This is nice for development since you do not need to rebuild the kernel each time you make some changes. Once the driver is loaded we will be able to export, configure and enable our PWM module.

``` bash
cd /sys/class/pwm/

echo 0 > pwmchip0/export 
echo 1000000 > pwm0/period 
echo 900000 > pwm0/duty_cycle
echo inversed > pwm0/polarity 
echo 1 > pwm0/enable 
```

## Summary

The figure below describes (roughly) how all the layers of abstraction are connected.

![_config.yml]({{ site.baseurl }}/images/hdl_to_linux_userland/overview.png)

We started off designing, verifying, and implementing an IP in the FPGA, provided the Linux Kernel with the details about the device using a device-tree node, and finally wrote and tested a PWM-Driver. Since this blog post was focused on interfaces and the general design approach, I did not provide the source code itself. However, I hope it was still interesting and you gained some insight into how I design and integrate custom IPs.