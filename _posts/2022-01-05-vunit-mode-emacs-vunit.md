---
id: 581
title: 'A brief overview of the Emacs VUnit-Mode'
date: '2022-01-05T14:59:42+00:00'
author: admin
layout: post
guid: 'https://embed-me.com/?p=581'
permalink: /vunit-mode-emacs-vunit/
wp_featherlight_disable:
    - ''
categories:
    - 'FPGA Development'
tags:
    - development
    - Emacs
    - HDL
    - VHDL
    - VUnit
    - vunit-mode
---

## Intro

Anybody who is working in HDL Design should know what [VUnit ](https://vunit.github.io/#)is, however, for those of you who do not, it is an excellent open-source testing framework for HDL. To be honest, I cannot imagine verifying code without it anymore. If you follow my blog, you might have noticed that I previously mentioned VUnit in [this posts](https://embed-me.com/from-hdl-to-linux-userland/) code verification section already. However, I constantly had to switch between a terminal to run unit tests and Emacs while developing. To get around this and create a more productive development environment, I decided to write an Emacs minor mode for VUnit. The upcoming short sections describe how the mode can be installed, how it operates, and what the future outlook is.

## The Mode

The mode acts as a global minor-mode and binds/hooks to the VHDL-mode. Once the mode is loaded, the default keybinding “`C-x x`” invokes the main menu, which looks as follows.

![_config.yml]({{ site.baseurl }}/images/emacs_vunit_mode/vunit_mode_menu.png)

The available options are pretty much self-explanatory, however, the colors have some special meaning that requires additional explanation. The ones marked in blue will *execute* the specified action *and then quit* the vunit-mode command window. The ones marked in red, however, will *add additional flags* for the actions available in blue. For example, in order to simulate the test case at the current cursor position in the GUI (eg. Modelsim) first enable the GUI flag by pressing `g` (the Minibuffer will display the currently enabled flags) and then start the simulation by pressing `t`.

Here is a quick demo from the [official vunit-mode Github repository](https://github.com/embed-me/vunit-mode) that shows some use-cases.

![_config.yml]({{ site.baseurl }}/images/emacs_vunit_mode/animation.gif)

Note that, if you run a command for the first time and you have not configured the path to the VUnit directory, you will be prompted for it. It is also possible to change the path at any point in time using the `vunit-get-path` function.

## The Setup

Since the package is part of [MELPA](https://melpa.org/#/), the installation is pretty simple and can be done within `package-list-packages`. After that enable the mode in your emacs configuration file (eg. init.el) with the following code snippet:

``` lisp
(require 'vunit-mode)
```

In order to customize the mode, simply run `M-x customize`, search for vunit and the following configuration should appear.

![_config.yml]({{ site.baseurl }}/images/emacs_vunit_mode/vunit_mode_customize.png)

In order for the VUnit-Mode to work, install VUnit according to the [official documentation](https://vunit.github.io/installing.html) and ensure that your preferred simulator is in the path.

## What’s next

At this very early stage, the mode is already capable of handling the most basic tasks, but I would really love to get some feedback and ideas from the community to drive development even further. If you face any problems, questions, or would like to participate in the project, please let me know. Ah, and by the way, happy new year 😉