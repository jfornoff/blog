---
date: "2018-09-10T19:53:51Z"
title: Introduction to the Transmission Control Protocol
tags:
- networking
---

This post continues the series on the fundamental protocols of the Internet, with a focus on the abstractions they provide and how they build on each other.

TCP (the Transmission Control Protocol) builds on top of IP (find the [accompanying post here]({{< ref "2018-05-27-introduction-to-ip.markdown" >}}). Based on the best-effort delivery guarantees of IP, it adds **reliable byte stream guarantees**.

In plain english this means:

* Messages sent are eventually received (**Reliability**)
* Incidental reordering of arriving data fragments is resolved

TCP acknowledges the concept of inter-process communication (IP doesn't have that, it just cares about how to get messages from one **host** to another). It is fundamentally based on connecting a "pair of processes" (if you are interested, [RFC 793](https://tools.ietf.org/html/rfc793)).

Staying with the theme of providing services to upper layers, TCP offers:

1. Ensuring that the receiving process can keep up with the senders pace (**Flow control**)
2. Utilizing the underlying bandwidth without overloading the network (**Congestion control**)
3. Ordering (using ascending sequence numbers attached to each data segment)
4. Reliability by data retransmission

These properties are what make TCP so popular, few people really want to think deeply about what happens when data segments are lost or reordered on the path over the network.

#### Bird's eye view
For the rest of this post, I'd like to give you an impression how TCP is actually used by applications or upper-level protocols.

To start communicating with another process, it is necessary to establish a TCP connection.
The end product of this is a connected pair of **Sockets**, essentially an abstraction allowing to read and write data through the connection (you can check out my post on [Linux Sockets]({{< ref "2018-01-20-introduction-to-linux-sockets.markdown">}}) if you want to know the details).
Symmetrically to the setup, there is a teardown sequence that is performed when you [`close`](https://linux.die.net/man/3/close) a Socket on either side.

#### Buffering
The TCP implementations on both sides of the connections are keeping buffers for both transmission directions -- _receiving_ and _sending_.
These are limiting the amount of data that can be:

- sent without the other side consuming it
- received without being consumed

Applications send and receive data using **system calls** (i.e., your application walks to the Linux kernel and asks it to [`send`](https://linux.die.net/man/2/send) or [`recv`](https://linux.die.net/man/2/recv) from the network). In between [`recv`](https://linux.die.net/man/2/recv) calls, the Kernel buffers a finite amount of data in the Socket **receive buffer**. If that buffer runs full, TCP's **flow control** mechanism allows indicating to the sending application to stop sending for now. The technical term for this sort of behavior is **back pressure**. The interesting part about this is that the sending application doesn't need to know about this behavior, the [`send`](https://linux.die.net/man/2/send) system call just starts blocking until the data can be sent. [`recv`](https://linux.die.net/man/2/recv) behaves similarly on an empty receive buffer -- it just blocks until it has some data to deliver.

#### Conclusion
Summing up, TCP abstracts away the complications of dealing with an unreliable underlying network.
Using these provided services, applications and upper layers can just focus on sending and receiving data over a reliable channel.
