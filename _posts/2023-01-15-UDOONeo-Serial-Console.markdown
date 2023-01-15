---
layout: post
title:  Getting serial on UDOONeo Board
date:   2023-01-15
categories: cryptography, network security, c++
---

UDOONeo is a single core I.MX6 Cortex-A9 board with peripherals such as CAN and Ethernet on the base model.

Firmware image can be purpose build with yocto which was too complex / never be built successfully on any version of linux.

Serial console is something that i looked to port a new version of software on to it.

UDOONeo webpage lists the serial console ports on UART1 -> TX and RX, GND can be connected anywhere.

Below is what i setup with the UDOONeo. First two pins go to RX and TX pins on the Serial breakout board.

GND pin should go to GND on the breakout. No need to supply power so VCC is not connected.


Set baudrate 115200 , 8N1 on minicom. With this, the console is accessible.

![image](_posts/IMG-3881.jpg)
