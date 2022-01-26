---
layout: post
title:  "defending against port scanners - part 1"
date:   2022-01-16 01:04:00
categories: cryptography, network security, c++
---

I have been working on a personal IDPS project during the free time for learning and research purposes. This below article is one of the outcomes. The article primarily focuses on defending against the TCP port scanner with SYN usecase.

Port scanners are one of the primary methods to find out open ports on a device that is attached to the network. These open ports further can be used in the process of finding vulnerabilities in the device. Further port scanners would exploit OS specific default configuration parameters for the networking stack and they identify the OS based on these (signatures).

Since port scanners run at L4, generally TCP, UDP and ICMP are used to perform port scanning. Random number of requests are made for every port between 1 and 65535 and the scanner expects a response back, based on the response the port scanner would then determine that a set of services are open.

For example, in case of TCP, the port scanner initiates connection requests with SYN bit set to the server ( in the TCP flags). In general, according to the protocol, if there is no service available, the server would send out a RST (or in some cases RST + ACK). If the server sends out a SYN + ACK (service available), then the scanner understands that service is available.

For example a series of SYN and RST are shown below.

See that in the above capture file, the SYN bit is set in the flags from `192.168.75.1` (scanner) to target `192.168.75.132`. The response shows a RST + ACK, meaning the unavailability of the service.

![Capture_File](https://raw.githubusercontent.com/madmax440/madmax440.github.io/master/_posts/Screenshot%20from%202022-01-26%2004-33-07.png)

This is one example, but there are many otherways to identifying open TCP ports. For now, lets look at how this can be prevented for the SYN usecase.

An IDPS generally will have 3 stages.

1. parsing
2. filtering
3. reporting / alerts / events

-> Parsing involve decoding packets, uncovering all the layers upto the application.

-> Filtering involve a task of handling filters at each layer. Filters are applied on the set of rules to match incoming packet. Sometimes, identifiers in the packets are then used to track the state of the connection / communication. (Connection tracking)

-> reporting / alerts / events are generated as the output of the filter. Events are formatted in a way / representation for monitoring and further analysis.

Considering the TCP SYN case for port scanning, the below diagram shows 

![Capture_File](https://raw.githubusercontent.com/madmax440/madmax440.github.io/master/_posts/Untitled%20Diagram-TCP%20filtering%20-%201(1).jpg)

1. An incoming packet is parsed by the decoder.
2. packet is validated for the ipv4 / ipv6 ethertype.
3. ipv4 / ipv6 headers are parsed. validated for consistency (checksum, header fields etc)
4. validate if ipv4.protocol == TCP or ipv6.next_header == TCP.
5. perform TCP header decode. validate for consistency (checksum, header fields etc)
6. if checksum is valid, perform the packet filtering for TCP.



At each stage, if the packet checks fail, the packet is dropped and an alert is generated with corresponding failure.

