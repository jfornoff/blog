+++
title = "Consistent Hashing"
description = "The what, why and how"
date = 2019-07-14T10:00:00+02:00
tags = ["computer science", "algorithms"]
draft = false
+++

# The Problem
You need to handle more requests than a single server can deal with.

* First thought: Well, we can have **two servers**, duh.
* Second thought: How do we decide where each request should go?

There are different methods of distributing requests between servers.
The simplest method that intuitively comes to mind is **Round Robin**, where you allocate requests
to servers one-by-one, starting over at the beginning when everyone had a turn.
Obviously, the distribution of requests is very even (note that this does not quite imply perfect utilization, since not every request takes equal resources and time to process).

<center>
  {{< figure src="/assets/img/consistenthashing/roundrobin.svg" class="fill" caption="Round Robin" >}}
</center>

This doesn't always work great.
Imagine a case where you have to do slow computations to serve requests. You could cache the results to save redundant work and make response times faster.
In this case, applying Round Robin would lead to redundant work and inefficient memory usage (due to each server computing and caching results).

Now imagine load increasing to a point where the servers must expire cache entries due to limited memory space.
These expired cache entires will have to get recomputed or fetched, again doing avoidable work.

Here's where request hashing can be useful.

# Request hashing

A hash function (in the context of distribution) is not much more than a mathematical formula that projects a large space of values into a smaller one. This method lends itself well to mapping requests to servers (or "items" to "buckets", which is essentially equivalent).
This mapping is also stable, i.e., two identical requests are going to have the same hash value, thus mapping to the same server.

Load balancing can exploit this property to choose a hash function such that requests requiring the same cache data hit the same servers. When more similar requests hit the same servers, we increase the likelihood of cache hits! 🎉

For an example, let's pretend that we have a service that could profit from caching, and we have three servers.
<center>
  {{< figure src="/assets/img/consistenthashing/loadbalancingexample.svg" class="fill">}}
</center>

We could then use a hash to distribute requests naively as follows:

{{< highlight go >}}
ServerToSelect(Request, NumServers) = Hash(Request) mod NumServers
{{< / highlight >}}

Let's also say we have three requests hashing as follows:
{{< highlight go >}}
Hash(R1) = 3
Hash(R2) = 8
Hash(R3) = 19
{{< / highlight >}}

This would give us:
{{< highlight go >}}
ServerToSelect(R1, 3) = 0 // first server
ServerToSelect(R2, 3) = 2 // third server
ServerToSelect(R3, 3) = 1 // second server
{{< / highlight >}}

Now let's say we look at our monitoring and notice that an extra server should be added.

### Ohhh no, elasticity!

Let's see how the additional server affects the request allocation.

{{< highlight go >}}
ServerToSelect(R1, 4) = 3 // fourth server
ServerToSelect(R2, 4) = 0 // first server
ServerToSelect(R3, 4) = 3 // fourth server
{{< / highlight >}}

As we can see, after adding a node, the request allocation totally changed.
This is problematic since all the cached data on the servers is now either useless
or at least less relevant to the requests hitting it.

Can we do better than this? Well of course we can!

# Consistent hashing
Consistent hashing provides us a more stable way of dividing requests in the face of changing
server pools (original paper for those interested: [1](https://www.akamai.com/us/en/multimedia/documents/technical-publication/consistent-hashing-and-random-trees-distributed-caching-protocols-for-relieving-hot-spots-on-the-world-wide-web-technical-publication.pdf)).

Here is how it generally works:

1. We hash an identifier of each server in our server pool (e.g., their IP).
{{< highlight go >}}
Hash(S1.IP) = X
Hash(S2.IP) = Y
Hash(S3.IP) = Z
{{< / highlight >}}

2. We divide the Hash space between all servers (we'll see how that works in the next section).
3. We hash requests, the server who "owns" that part of the hash space fulfills it.


### The Visualization

<center>
  {{< figure src="/assets/img/consistenthashing/hashring.svg" class="fill" >}}
</center>

Don't freak out yet. This is a "Hash Ring" and acts as a visual way of reasoning about how requests are going to map to servers.
We are going to assume that our Hash function returns a 64-bit integer.
The ring forms by "wrapping around" from the maximum value (`2^64 - 1`) to the minimum value (`0`).

To determine the responsible server for a request, we look at the hash value for said request.
Then we determine the "closest" server on the hash ring.

Implementation may work in different ways:

* Start from the Request hash value, go clockwise until encountering a Server hash value
* Compare absolute distances to the two closest Server hash values (in either direction)

For example:
When looking at <span style="color: #03cc01">**R3**</span>, absolute distance would map it to **Server 3**, clockwise rotation would map it to **Server 2**.

The important thing to realize is that every server "owns" a part of the hash ring,
i.e., a range of request hash values that will map to it.

Upon **adding** a server, it takes over a part of the hash ring from neighboring servers.
Upon **removing** a server, neighboring servers are taking over the part of the removed server.
The most important part: most of the hash ring (and all request mapping therein) remains unaffected.

This unlocks the benefit that servers can come and go without large impacts on request distribution.
Furthermore, this distribution technique handles limited views of the server pool well.
If a server is unknown to some peers, e.g., when it has just joined the server pool, some requests will still hit the server that "owned" that section of the hash ring before. However, there is a high probability that it will still have warm caches for these types of requests!

# Conclusion
Consistent hashing allows for more robust reassignment semantics when spreading requests
across a dynamic set of servers. While slightly more complicated to implement, it's still fairly simple.

Closing off, I want to note two more things.
Firstly, dividing the hash ring space **evenly** between all servers is not inherently solved by consistent hashing. One approach that alleviates the problem is replicating servers to multiple points on the hash ring, this comes with the drawback of multiplying memory usage.

Lastly, the problem considered above is **web caching**. The distribution provided by consistent hashing also proves useful
for distributed storage systems (i.e., storage systems where you cannot fit the data set onto a single machine).
The division of a dataset between computers is then called *sharding*. Adding and removing shards benefits from consistent hashing as well, since it minimizes the data that needs to transferred between shards to keep the data base consistent. (Interesting papers: [2](https://arxiv.org/abs/1406.2294))

But this is a topic for another post. Bye!
