---
layout: post
title:  Detailing and Packaging IDS Events
date:   2022-09-20
categories: cryptography, network security, c++
---

## Detailing / Format of the IDS Event

IDS events generally have to provide with a detailed information of where and when the event has occured, what type of event has occured and if there's a rule that's matched if the detection mechanism is signature based.

Where is solved by adding the IP address or Mac address. When is solved with the use of microsecond timestamp. What is solved with a detail of the event.

Lets look at a general C structure of the event that can be described.

```c
struct idps_event {
    idps_event_type type;
    idps_event_detail detail;
    uint32_t ts_sec;
    uint32_t ts_usec;
    uint32_t rule_id; // matched rule set to 0xFFFFFFFF if no rule matched
    uint32_t flags;
    
    union {
    
      uint16_t ethertype;
      
      union {
          uint8_t sender_ip[16];
          uint8_t target_ip[16];
          uint16_t protocol;
          
          union {
              uint16_t src_port;
              uint16_t dst_port;
          } l4;
      } l3;
    } eh;
    uint16_t src_port;
    uint16_t dst_port;
    uint16_t pkt_len;
    uint8_t pkt[1500];
};
```

Above structure defines the two enumerations. `idps_event_type` and `idps_event_detail`.

`idps_event_type` maps to the action done by the IDS.

```c
typedef enum idps_event_type {
    ALLOW,
    DENY,
    NOTIFY,
};
```

`idps_event_detail` maps to what type of an event that has occured. For example,

```c
typedef enum idps_event_detail {
    INVALID_ETHERTYPE,
    INVALID_IP_CLASS,
    INVALID_PORT_RANGE,
};
```

The fileds `ts_sec` and `ts_usec` allow the representation of when the event occured in UTC.

The field `rule_id` describe what rule the IDS has hit. if no rule is matched, one can set it to an invalid number. Signature based detections often involve matching against known signatures. Most likely that the signatures need to be updated in order to get a sane `rule_id`.

Rest of the fileds represent a packet in general. One can argue that if a full offending packet is sent, then there is no need of sending these network parameters. It is true however, if one wants to minimize network bandwidth, they can opt in only few elements instead of sending an entire packet.

Ofcourse not all packets are application. Some are only ip level routing transactions or tcp level connections. In such cases, flags field is used to denote which fields are present to further cut down on the packet size.

A typical ipv4 event that includes protocol will look like below

```bash
| 1 byte     | 4 bytes        | 4 bytes | 4 bytes | 4 bytes | 2 bytes   |                      4 bytes                     |
|            |                |         |         |         |           | 7    |   6  |   5       | 4 | 3 | 2 | 1 | 0 | .. |
| event type | event details  | ts_sec  | ts_usec | rule_id | ethertype | ipv4 | ipv6 |  protocol | reserved            .. |
|--------------------------------------------------------------------------------------------------------------------------|
| 4 bytes    | 4 bytes     | 2 bytes  |
| src_ipaddr | dest_ipaddr | protocol |
|-------------------------------------|
```

Size allocation on the wire or on file system is 33 bytes, which is ok'ish. Some optimizations can be done further but that depends on type of system.

On a small scale embedded system, one could reduce 2 bytes on event_details and 2 bytes on rule_id, as there wont be large number of events beyond 65535 and as well not many rules beyond 65535. Further down flags can be cut down to 16 bits, thus saving 6 bytes resulting in 27 bytes.

On a unconstrained system, one could simply use the same format and package them in a json. Reasons are explained below in packaging IDS events.

For example, a json based event can be expressed as follows,

```json
{
    "event_type": "deny",
    "event_description": "invalid protocol specified in ipv4 header",
    "timestamp": {
      "sec": 848147888,
      "usec": 124828141
    },
    "rule_id":  41,
    "ethertype": "0x800",
    "flags": {
      "ipv4": true,
      "ipv6": false,
      "protocol": false,
    },
    "ipv4": {
      "src_ipaddr": "192.168.1.4",
      "dst_ipaddr": "192.168.1.2",
      "protocol": 4919
    }
}
```

## Packaging IDS Events 

There are in general various ways to provide IDS events. Some update the events periodically over the network, some store the events in a file and based on a threshold they upload the events over the network.

Generally it depends on the configuration of the system and placement of the firewall device in the network, hardware storage, CPU Resources and network bandwidth use.

So here are few of the needs / requirements regards to the events.

1. Users would want to use the received events for further review of the events / alerts happened.
2. USers would want to see the events in a dashboard.

So in general one could use `json` format and pack the event into a json and pass it to the server over TLS. This seem very good approach and so the Users could directly visualize by reading the json logs or view the event in the dashboard.

However json tend to be very ineffcient when it comes to network bandwidth. Thus we come across few of the serialization formats such as `ASN.1` or `protobuf`. Sometimes we write our own serialization format packaged into byte buffer for upload.

Choosing the right format of the data upload saves the following:

1. Storage space (if events need to be stored in RAM or in Flash)
2. Network bandwidth usage on metered internet connections - such as for example a Car that collects events within the vehicle, IoT device etc.

Below is the table illustration of the pros and cons on a single core Cortex-A7 based systems might look like this.

| Messaging format | RAM use | Flash use | cpu cycles | 
|------------------|---------|-----------|------------|
| Json | heavy | heavy | moderate |
| protobuf| moderate | minimal | heavy |
| ASN.1 | moderate | minimal | moderate |
| binary | minimal | minimal | minimal |

Sometimes the availability of the libraries for protobuf and ASN.1 makes a big hurdle to cross on few embedded systems.

Protobuf, ASN.1 and Json suffer the serialization and use of RAM to serialize the message. However Protobuf and ASN.1 could reduce the event size.


## Conclusions

1. Choose the type of format based on the system being used - unconstrained vs constrained.
2. Choose based on the availability of serializers - binary, protobuf, flat buffers, ASN.1 and so on.

