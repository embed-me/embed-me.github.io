---
title: 'Building a KNX-based UP Buzzer for Smart Home Notifications'
permalink: /knx-up-buzzer-smarthome/
categories:
    - 'Development'
tags:
    - development
    - C++
    - KNX
    - 'Home Automation'
---

## Intro

In this post, I will walk you through the process of building a simple KNX-based buzzer system designed to notify you of events. The project leverages the KNX stack from [thelsing](https://github.com/thelsing/knx) and [Arduino](https://github.com/arduino/) on the driver layer, and integrates it with an RP2040-based MCU connected to the KNX bus using a BCU (Bus Coupling Unit). The system is designed for customization, with the option to extend the functionality by adjusting configuration files and behaviors.

The entire project is available on [GitHub](https://github.com/embed-me/knx-up-buzzer/tree/main), where you'll find all the necessary code, configuration files, and instructions on how to build your own KNX buzzer.

## What it offers
The UP Buzzer offers flexibility in how it can be used, with support for up to 8 configurable melodies. 
Each melody can be individually configured with one of three modes. 
These modes determine how and when a melody plays, giving you a variety of options for different notification scenarios described below.

Detailed configuration screenshots and more information are available on the [GitHub repository](https://github.com/embed-me/knx-up-buzzer/tree/main/software).

### Trigger Mode
In Trigger Mode allows you to play a melody in a one-shot fashion. This is for example great to get feedback in case the washing machine or dryer is finished.

![Trigger Mode](https://github.com/embed-me/knx-up-buzzer/blob/main/software/img/trigger.png?raw=true)

### Switch Mode
In Switch Mode, the melody is played continuously until a stop command is received. This mode is ideal for ongoing alerts or notifications, for example during the pending state of an alarm until you enter the code to disable it.

![Switch Mode](https://github.com/embed-me/knx-up-buzzer/blob/main/software/img/switch.png?raw=true)

### Venting Monitor Mode
This mode tracks the duration of opened windows or doors in combination with the outside temperature to notify you in case the venting time gets exceeded. It can come in handy as a simple reminder to close opened windows or doors in extreme weather conditions.

![Venting Mode](https://github.com/embed-me/knx-up-buzzer/blob/main/software/img/venting_mode.png?raw=true)

## Hardware/PCB

Although I made use of some already available PCBs for the MCU and the BCU, the PCB for the buzzer PCB I designed by myself. Currently only the buzzer is in use, however, I also added a circuit for a proximity sensor and binary inputs that could potentially be supported in the future. For now they are untested and not mounted.

<img src="https://github.com/embed-me/knx-up-buzzer/blob/main/hardware/img/pcb_3d_view.png?raw=true" alt="assembly" width="40%" height="auto">

For manufacturing I can really recommend [JLCPCB](https://jlcpcb.com/)! They are very quick, dirty cheap and the quality for such a trivial board as this is more than sufficient. Taking into consideration the discount I payed a staggering $3.58 for 10 pcs including shippment - incredible!

## Firmware

The firmware build is configured using [PlatformIO](https://docs.platformio.org/en/latest/what-is-platformio.html), an vendor independent, cross-platform tool to build embedded products. The code is designed to run on the RP2040-based board, and all necessary libraries and dependencies are managed via PlatformIO and automatically loaded to make the build as convinient as possible.

For detailed steps on how to build the project or modify it see [GitHub Firmware](https://github.com/embed-me/knx-up-buzzer/tree/main/firmware).

## Housing

A custom 3D-printed enclosure houses the essential components. The main challenge was to get the form factor of both components right and ensuring that the audio feedback is loud enough even if enclosed. By assemling all components virtually before printing it was possible to design them well enough for a perfect fit.

<img src="https://github.com/embed-me/knx-up-buzzer/blob/main/housing/img/housing_asm_transparent_top.png?raw=true" alt="assembly" width="40%" height="auto">

The printed and assembled housing was actually quite nice for a cheap 3D print and fulfilled all requirements from above.

<img src="https://github.com/embed-me/knx-up-buzzer/blob/main/housing/img/up-buzzer-final.png?raw=true" alt="3Dprint" width="40%" height="auto">

The STL files for 3D printing the housing are available in the GitHub repository. Feel free to customize and print your own.

## Software Configuration

The really nice part is that the UP Buzzer is completely configurable in [ETS](https://www.knx.org/knx-en/for-professionals/software/ets6/). An open source tool called [Kaenx-Creator](https://github.com/OpenKNX/Kaenx-Creator) was used to generate the signed product database which can then be imported to ETS. In addition Kaenx-Creator simplifies the relation to the firmware because it automatically generates a header file to access configurations from storage, group object numbers and more which is much more convinient than maintaing the file yourself.

Detailed steps on how to configure and generate the product database can be found on [GitHub Software](https://github.com/embed-me/knx-up-buzzer/tree/main/software).

## Conclusion

The UP Buzzer is a simple yet powerful solution for adding smart home notifications using the KNX protocol. By combining Kaenx-Creator for configuration, PlatformIO for development, and the flexibility of KNX modes, this project enables you to create a customizable buzzer system for various automation tasks.

For more details, including the source code, configuration files, and step-by-step guides, visit the [GitHub repository](https://github.com/embed-me/knx-up-buzzer/tree/main). This project can be easily extended to suit your specific needs, whether it's for alerts, notifications, or other smart home use cases.