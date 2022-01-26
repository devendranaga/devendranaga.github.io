---
layout: post
title:  "defending against port scanners - part 1"
date:   2022-01-16 01:04:00
categories: cryptography, network security, c++
---

Introduction:
=============

I have been working on a personal IDPS project during the free time for learning and research purposes. This work in progress article is one of the outcomes. The article primarily focuses on defending against the TCP port scanner with SYN usecase.

Port scanners are one of the primary methods to find out open ports on a device that is attached to the network. These open ports further can be used in the process of finding vulnerabilities in the device. Further port scanners would exploit OS specific default configuration parameters for the networking stack and they identify the OS based on these (signatures).

Since port scanners run at L4, generally TCP, UDP and ICMP are used to perform port scanning. Random number of requests are made for every port between 1 and 65535 and the scanner expects a response back, based on the response the port scanner would then determine that a set of services are open.

Port scan detection:
====================

For example, in case of TCP, the port scanner initiates connection requests with SYN bit set to the server ( in the TCP flags). In general, according to the protocol, if there is no service available, the server would send out a RST (or in some cases RST + ACK). If the server sends out a SYN + ACK (service available), then the scanner understands that service is available.

For example a series of SYN and RST are shown below.

See that in the above capture file, the SYN bit is set in the flags from `192.168.75.1` (scanner) to target `192.168.75.132`. The response shows a RST + ACK, meaning the unavailability of the service.

![Capture_File](https://raw.githubusercontent.com/madmax440/madmax440.github.io/master/_posts/Screenshot%20from%202022-01-26%2004-33-07.png)

This is one example, but there are many otherways to identifying open TCP ports. For now, lets look at how this can be prevented for the SYN usecase.

An IDPS generally will have 3 stages.

1. parsing
2. filtering
3. reporting / alerts / events

1.) Parsing involve decoding packets, uncovering all the layers upto the application.

2.) Filtering involve a task of handling filters at each layer. Filters are applied on the set of rules to match incoming packet. Sometimes, identifiers in the packets are then used to track the state of the connection / communication. (Connection tracking)

3.) reporting / alerts / events are generated as the output of the filter. Events are formatted in a way / representation for monitoring and further analysis.

Considering the TCP SYN case for port scanning, the below diagram shows (zoom into the picture for high quality flow chart) the high level flow of sequence.

![TCP_decode](https://raw.githubusercontent.com/madmax440/madmax440.github.io/master/_posts/Untitled%20Diagram-TCP%20filtering%20-%201(1).jpg)

1. An incoming packet is parsed by the decoder.
2. packet is validated for the ipv4 / ipv6 ethertype.
3. ipv4 / ipv6 headers are parsed. validated for consistency (checksum, header fields etc)
4. validate if ipv4.protocol == TCP or ipv6.next_header == TCP.
5. perform TCP header decode. validate for consistency (checksum, header fields etc)
6. if checksum is valid, perform the packet filtering for TCP.



At each stage, if the packet checks fail, the packet is dropped and an alert is generated with corresponding failure.

We run the port scan detection before we run the TCP connection tracking algorithm. This is mainly to avoid polluting the connection tracking tables and give enough space for tracking real hosts.

The TCP SYN based Port scanner defense algorithm is as follows.

1. If TCP header contains SYN, match against open ports table. (described later section)
2. if packet not matched aganist open ports, create a new temporary entry with src_ip, src_port, dst_ip, dst_port in a table. Allow the packet to the target.
3. Wait for any response back from the server that contain RST + ACK for the original packet. Match against the stored entry (server_pkt.dst_ip == stored_entry.src_ip && server_pkt.dst_port == stored_entry.src_port && servere_pkt.ack_no == stored_entry.seq_no).
5. Valid match, deny the packet.
6. Packet not matched, match against connection track table. A valid match found, allow packet to the destination. (Legit packet usecase) This can also be further passed down to the connection track algorithm and have it decide if the packet can be allowed.

Below diagram shows the detection process. (zoom in to the picture for high quality flow chart)

![Detection_method](https://raw.githubusercontent.com/madmax440/madmax440.github.io/master/_posts/Untitled%20Diagram-Detection%20of%20Port%20Scanner%20-1(1).jpg)

The above method mainly dependent on the RST+ACK behavior of the OS.

Once the packet is denied, store the following information about the packet.

1. src_ip
2. src_port
3. count of denials
4. connection attempts
5. average delta time between packets from the sender
6. port_scan_in_progress to false 

Run the below steps periodically or as and when the packets are being detected.

For each item in the table,

1. match connection attempts over a threshold of connection attempts (configurable).
2. match count of denials against configured value.
3. match delta_timestamp against a threshold of delta_timestamp between each packet (configurable).
4. if both 1, 2 and 3 are successful, set port_scan_in_progress. Raise an alert.

In case of unmanaged systems, open ports can be learnt by storing the allowed src_ip, src_port in the open_ports table. This can be further used at the input stage itself, to perform better detection.

Few observations:
=================

1. Sometimes packet scanners might defeat the delta_timestamp thresholds by delaying and spreading the attack over a wide time range. Future work will append more cases to the port scanning detection algorithm.

2. Another Problem is the table overflow when attacker runs over many ip addresses, but this can be overcome with having fixed incoming connections and enabling the SYN cookies.

3. Having fixed set of state tables avoids overruns of the available system memory. This shall be determined based on available system memory, size of the network, number of incoming requests and so on.


It is always better to temporarily deny the __offending__ src_ip and src_port. It could be because a compromised network hardware could also been a potential port scanner. At this moment i have not yet figured the technique to unblock the sender. It can either be configurable, or could be presented to the system admin and have them unblock manually after further analysis.

The code for this technique will be published sooner..
