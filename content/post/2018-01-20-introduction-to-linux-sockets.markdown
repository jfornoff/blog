---
date: "2018-01-20T11:14:58Z"
title: Introduction to Linux sockets
tags:
- networking
---

This post will cover a little bit about the abstraction of a Socket in Linuxworld, what they exactly are and how they are generally used.

---
## What are Sockets and how are they created?

Sockets in the most general sense are an abstraction of interprocess communication. They represent the endpoint of a communication channel (abstracting the details of the underlying transport).

To interact with Sockets, you use system calls. Those are commands that you send the Linux kernel who manages the communication and networking stack.

Sockets are created by using the `socket()` system call.
Let's deconstruct this call by looking at it's arguments.

{{< highlight c >}}
int socket(int domain, int type, int protocol)
{{< /highlight >}}

# Communication domain

The communication domain specifies a family of protocols that this Socket is going to live in.
Most prominently, Sockets can be "scoped" to the local machine (Unix socket) or the internet.

- UNIX domain (identified by the `AF_UNIX` constant)
- IPv4 internet (`AF_INET`)
- IPv6 internet (`AF_INET6`)
- a bunch more, which can be found in [the man page](http://man7.org/linux/man-pages/man2/socket.2.html)

# Socket type
The Socket type specifies the communication semantics which are going to be in effect for this socket.
Let's go through the most common examples:

#### Stream sockets (`SOCK_STREAM`)
Stream sockets provide what is called a *sequenced, reliable, two-way, connection-based byte stream*.
Deconstructing this in terms of guarantees provided:
- **Sequenced**: Messages are received in the same order in which they were sent
- **Reliable**:
  - No duplication: The same message is received at most once
  - No creation: Messages are not received unless sent (there are no random messages appearing out of thin air)
  - Validity: When a process sends a message to another one (given that they're correct and not buggy),
    it will eventually be received

- **two-way**: Both sides are able to send and receive
- **connection-based**: A session is established prior to data transfer

The protocol in the Internet communication domain for this socket type is TCP.

#### Datagram sockets (`SOCK_DGRAM`)
A datagram is:
> A self-contained, independent entity of data carrying sufficient information to be routed from the source to the destination computer without reliance on earlier exchanges between this source and destination computer and the transporting network.
> [RFC1594](https://tools.ietf.org/html/rfc1594)

Datagram sockets provide sending of *connectionless, unreliable messages of a fixed maximum length*.
The protocol in the Internet communication domain for this socket type is UDP.

There are more types of sockets, which can be found in [the man page](http://man7.org/linux/man-pages/man2/socket.2.html)

# Socket protocol
In most cases, the `int protocol` is redundant, it is used when the communication domain has more than one protocol available for use (for example on `SOCK_RAW` type sockets). Otherwise it can be nulled and ignored.


---
## Connecting Sockets
Obviously, Sockets are only useful when there is another program on the other end of the communication channel.
So how do Sockets connect to each other?

Most commonly, this happens by a Socket connecting to another Socket that is listening.
Let's step through this by a TCP client-server example.

#### Server side
When a Server is starting up, it:
1. Instantiates a Socket through the [`socket()`](https://www.freebsd.org/cgi/man.cgi?socket(2)) system call that we just saw.
2. Binds it to a name (in this case IP address and port) through [`bind()`](https://www.freebsd.org/cgi/man.cgi?bind(2))
3. Allows incoming connections on the Socket by calling [`listen()`](https://www.freebsd.org/cgi/man.cgi?listen(2))
4. The socket is now able to be connected to, essentially putting incoming connections into a queue
5. Calling [`accept()`](https://www.freebsd.org/cgi/man.cgi?accept(2)) pulls connections from the queue and allocates a new Socket for them which is returned

#### Client side
When trying to establish a connection to a Server, the steps are:
1. Instantiating a Socket through `socket()`, just like in the Server
2. Calling [`connect()`](https://www.freebsd.org/cgi/man.cgi?connect) with the name / address that the Server socket was bound to (in `SOCK_STREAM` sockets this attempts to initialize the session with the Server)
3. Sending some data (e.g., the HTTP request) to the server by using [`send()`](https://www.freebsd.org/cgi/man.cgi?query=send) or similarly [`write()`](https://www.freebsd.org/cgi/man.cgi?query=write) (which behaves the same, but you cannot set flags)
4. Reading a response using [`recv()`](https://www.freebsd.org/cgi/man.cgi?query=recv) or [`read()`](https://www.freebsd.org/cgi/man.cgi?query=read&sektion=2) - this call blocks (does not return) waiting unless the socket is set to use non-blocking I/O (by using [`fcntl()`](https://www.freebsd.org/cgi/man.cgi?query=fcntl))
5. When done interacting with the Server, [`close()`](https://www.freebsd.org/cgi/man.cgi?query=close) is called on the Socket, which deletes it and closes the established connection.

---
## Summing up
Hope this has taught you a thing or two about Sockets and given you a bit of insight on how Sockets actually work close to the metal.

Until next time!
