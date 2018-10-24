---
date: "2018-05-27T19:05:48Z"
title: Introduction to the Internet Protocol
tags:
- networking
aliases:
- /networking/2018/05/27/introduction-to-ip.html
---

The Internet Protocol (IP) is a crucial underpinning of the Internet as we take it for granted nowadays.
To be completely honest with you, there is two IP protocols (there is the "old version", IPv4 and the "new version", IPv6). Both are standardised in the form of RFCs ([IPv4](https://tools.ietf.org/html/rfc791) and [IPv6](https://tools.ietf.org/html/rfc8200)). There are many more RFCs specifying details of the technologies, augmenting the base RFCs.

However, this post is _not_ about technical details.

#### What this post is not about
* Differences between IPv4 and IPv6 (*Spoiler:* v6 is pretty damn awesome.)
* How protocol headers look like \\
  (It's interesting but you can do it yourself, it's just information in a predefined format.)

#### What this post IS about
Instead, this post goes over the Internet Protocol on a very high and abstract level: \\
How IP fits into the Internet protocol stack, the guarantees that it provides and how the OSI layers separate concerns.

## Service: Giving and taking
IP resides at the **Network Layer (Layer 3)** of the OSI stack.
The core idea of these layers is __providing services__ to an upper layer, and __using services__ provided by a lower layer.


Lower layers know __nothing__ about how their services are used by upper layers. \\
Upper layers know __nothing__ about how exactly lower layers operate. \\
It does not concern them either -- it's an agreed-upon contract of service delivery.

The service that IP provides is:

* Addressing nodes on a network
* Forwarding and routing packets between nodes on the network

In summary, IP provides upper layers with the service to send data to another node on the network without worrying about how to get the packet to the right place in the network topology.
The upper layer (usually the **Transport Layer**, but there are exceptions) may be for instance TCP or UDP.

IP in turn uses services of the **Data Link Layer (Layer 2)**, which provides delivery of data frames within e.g., a LAN (local-area network) or WAN (wide-area network).
For the former, services on this level may be Ethernet or Wireless LAN.

As such, each layer has a well defined responsibility, i.e.:

**Transport Layer** knows about strategies of connecting pairs of processes (e.g., your web browser and a server) -- it may provide e.g., persistent connections, reliability and flow control.

**Network Layer** knows how to navigate data from A to B within a network (the Transport Layer is completely oblivious of how to do that, it just assumes it can address someone and the packet will -- most likely -- make it there).

**Data Link Layer** knows how to transport the data between nodes in a network (e.g., moving data between your Laptop and Home Router using WiFi or Ethernet).

## Guarantees
Now IP is a so-called **best-effort protocol**. Plainly stated, this means: _"I can't promise your packet will make it, but I'll try, ok?"_. Sending an IP datagram therefore does not guarantee that it arrives at its destination.

Sending may fail for a multitude of reasons:

* Very busy Routers on the network path being forced to discard packages (also called "Congestion")
* The underlying physical medium causing the package to be delivered incompletely
* Packages being intentionally discarded by e.g., Firewalls or Routers

IP does not attempt any correction in case this happens. This is decided by the upper layers (e.g., TCP can retransmit lost segments of its data stream when the delivery seems to have failed).

## Conclusion
This post attempts a new style of writing that focuses more on the broad architectural strokes of what the Internet is made up of. There is a lot of material on technical details. However, I personally feel like intuition for the underlying thought framework is way harder to find.

Hope you found this useful, until next time.
