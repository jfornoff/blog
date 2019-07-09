+++
title = "Consistent Hashing"
description = "The what, why and how"
date = 2019-04-11T18:53:35+02:00
tags = ["computer science", "algorithms"]
draft = false
+++

# The Problem
You need to handle more requests than a single server can deal with.

* First thought: Well, we can have **two servers**, duh.
* Second thought: How do we decide where each request should go?

This problem is commonly known as **load balancing**.
There are different methods of distributing requests between servers, more often than not you want this
distribution be *even* to not overwhelm a single server.

The simplest method that intuitively comes to mind is **Round Robin**, where you allocate requests
to servers one-by-one, starting over at the beginning when everyone had a turn.
Obviously, the distribution of requests is very even (note that this does not quite imply perfect utilization, since not every request takes equal resources and time to process).

<center>
  {{< figure src="/assets/img/consistenthashing/roundrobin.svg" class="fill" caption="Round Robin" >}}
</center>

Unfortunately, this doesn't always work great.
Imagine a case where you have to do slow computations to serve requests. You could cache the results to save reduntant work and make response times faster.
Applying Round Robin there would lead to redundant work and inefficient memory usage (due to each server computing and caching results).

Now imagine load increasing to a point where the servers must expire cache entries due to limited memory space.
These expired cache entires will have to get recomputed, again doing avoidable work.

Here's where request hashing can be useful.

# Request hashing

A hash function (in the context of load balancing) is not much more than a mathematical formula that projects a large space
of values into a smaller one. This method lends itself well to mapping requests to servers.
This mapping is also stable, i.e., two identical requests are going to have the same hash value,
mapping to the same server.

Load balancing can exploit this property to hash requests such that those that require the same
cache data hit the same servers. When more similar requests hit the same servers, we increase the likelihood of cache hits! 🎉

For an example, let's pretend that we have a service that could profit from caching, and we have three servers.
<center>
  {{< figure src="/assets/img/consistenthashing/loadbalancingexample.svg" class="fill">}}
</center>

We could then use a hash to distribute requests as follows:

`ServerToSelect(Request, NumServers) = Hash(Request) mod NumServers`

Let's also say we have three requests hashing as follows:
```
Hash(R1) = 3
Hash(R2) = 8
Hash(R3) = 19
```

This would give us:
```
ServerToSelect(R1, 3) = 0 // first server
ServerToSelect(R2, 3) = 2 // third server
ServerToSelect(R3, 3) = 1 // second server
```

Now let's say we look at our monitoring and notice that an extra server should be added.

### Ohhh no, elasticity!

Let's see how the additional server affects the request allocation.
```
ServerToSelect(R1, 4) = 3 // fourth server
ServerToSelect(R2, 4) = 0 // first server
ServerToSelect(R3, 4) = 3 // fourth server
```
As we can see, after adding a node, the request allocation totally changed.
This is problematic since all the cached data on the servers is now either useless
or at least less relevant to the requests hitting it.

Can we do better than this?

# Consistent hashing

Well of course we can!

### The Visualization
- Hash ring, servers are put on the ring, incoming requests are mapped to points on the ring.
  -> Servers "own" part of the ring, e.g., everything in counterclockwise direction, until the next server's hash

Adding and removing resources:
Adding puts the server somewhere on the hash ring, which makes it take part of another server's hash space.
HOWEVER most requests will still hit the same servers!

# Be careful what you ~~wish~~ hash for

Don't stop thinking yet! While consistent hashing can help distribute work in a stable
fashion, it does not necessarily guarantee good load balancing.

Consider the following example:
Your webshop is scaling like crazy and have decided to partition your ML-powered pricing servers to cache results better.
Sounds great for consistent hashing, right? OK, let's throw a consistent hash on the product ID to split price requests across these servers.

You have a new hip product in stock that doesn't sell great so far.
Now the startup founders go on the "The Shark Tank" and totally kill it.
People start googling that product and your SEO is paying off, your shop is the first hit!

Your
