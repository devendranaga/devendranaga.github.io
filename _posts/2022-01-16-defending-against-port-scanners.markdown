---
layout: post
title:  "defending against port scanners - part 1"
date:   2022-01-16 01:04:00
categories: cryptography, network security, c++
---

I have been working on a personal IDPS project during the free time for learning and research purposes. This below article is one of the outcomes. The article primarily focuses on the TCP port scanner with SYN usecase.

Port scanners are one of the primary methods to find out open ports on a device that is attached to the network. These open ports further can be used in the process of finding vulnerabilities in the device. Further port scanners would exploit OS specific default configuration parameters for the networking stack and they identify the OS based on these (signatures).

Since port scanners run at L4, generally TCP, UDP and ICMP are used to perform port scanning. Random number of requests are made for every port between 1 and 65535 and the scanner expects a response back, based on the response the port scanner would then determine that a set of services are open.

For example, in case of TCP, the port scanner initiates connection requests with SYN bit set to the server ( in the TCP flags). In general, according to the protocol, if there is no service available, the server would send out a RST (or in some cases RST + ACK). If the server sends out a SYN + ACK (service available), then the scanner understands that service is available.

For example a series of SYN and RST are shown below.

See that in the above capture file, the SYN bit is set in the flags from `192.168.75.1` (scanner) to target `192.168.75.132`. The response shows a RST + ACK, meaning the unavailability of the service.

This is one example, but there are many otherways to identifying open TCP ports. For now, lets look at how this can be prevented for the SYN usecase.


