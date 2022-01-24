---
layout: post
title:  "defending against port scanners - part 1"
date:   2022-01-16 01:04:00
categories: cryptography, network security, c++
---

Few rules:
==========

### TCP based port scan defense:

if the originating syn packets are from the same host, then follow this method. However, the table building can be done in either case. (same or diferent host)

#### detection:

1. rate of pkts/sec from a given ip address cross a particular threshold.
2. condition 1, the host trying to connect to the server on various ports.
   1. match incoming port against the list of allowed ports on the server.
   2. if all ports are allowed (aka firewall not configured correctly), try to learn if there is RST from the server back.
   3. if server responds with an RST, make a state table about the list of the ports that are not open in the server.
   4. upon any new packet entry, match the packet against the learnt table and deny there itself.
4. more than x number of syn timeouts from the host.

#### prevention:

1. raise events to the administrator via different interfaces (logging, ui front end via mqtt etc)
2. blacklist host


if the originating syn packets are from various ip addresses, then follow this method.

1. limit number of incoming connections towards the server.
2. let the server enable syn cookies (not in control of the idps).

### UDP based port scan defense:

#### detection:

1. follow rate of pkts/sec logic stated in the tcp based defense.
2. detect the directed broadcast address on ipv4 destination. if present, deny the packet. (based on the class of the network we are in)
3. if the firewall rules set to allow specific udp ports - use it to deny early stage.
4. if it doesn't
    1. watch for icmp responses from the server with code destination unreachable.
    2. for every destination unreachable build a table.
    3. a new packet matching to that port comes in, deny it immediately
5. method 3 will not however, eliminate an active attack, however creates a cache for future fool proof attacks.
