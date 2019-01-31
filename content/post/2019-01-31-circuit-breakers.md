+++
title = "On Circuit Breakers"
description = ""
date = 2019-01-31T06:48:45+01:00
tags = ["circuit breakers", "fault tolerance", "reliability", "system design", "software development"]
+++

## Hit me baby, one more time
Circuit breakers are a software development pattern that aims to increase the reliability of a service in the face of unreliable external dependencies.

> If your system does not function when it has no access an external dependency, its reliability is necessarily the minimum reliability out of all its dependencies, or likely worse.

Circuit breakers rely on a property of failing requests: If there is a bunch of failing requests, subsequent requests are likely to fail as well. There are many reasons why that might be:

- The opposite service is down, crashed or overloaded
- Network connectivity is impaired
- Requests are rejected by the opposite site due to bugs or misconfigurations

All these conditions have in common that they probably won't be gone after a few milliseconds.
This impression is reinforced when we have tried a few times already and there is no recovery to be seen.

## Why the name?
As you likely know, a Circuit Breaker is an appliance used in electrical circuits to cut the connectivity in case of harmful conditions of very high current (as it often implies a short circuit).
It is providing **fault tolerance for electrical systems**. The Circuit Breaker pattern provides **fault tolerance for software systems**. So how does this concept transfer?

As we discussed above, if there is a bunch of errors when trying to interact with an external service, there is a point (e.g., after requests to the service failed consistently for some amount of time) at which we can assume that it is not terribly useful to even make the call anymore.
The Circuit Breaker is a software component that detects when this is the case and "pulls the plug". After that, it does not call the external service anymore. It just returns the error right away. Or even better, *there is a fallback strategy in place*.

## Meditating on failure remediation
The interesting part about circuit breaking is in my opinion, that you have to think about your failure recovery. What can you do if this or that external service fails to respond to requests?
Let's think about this on a little case study.

![System architecture](/assets/img/circuitbreakers/system.png)
On our shop homepage, we want to present some recommendations from products that the user might be interested in. The front-facing application queries a recommendations system. This system uses previous orders made to figure out suitable recommendations.

![Purchase system failure](/assets/img/circuitbreakers/purchasefailure.png)
Someone tripped over the power cord of the purchase system, we can't retrieve the previous orders anymore D:

How do we deal with this failure mode? We could throw our hands in the air in panic and stop serving requests altogether. Doesn't sound great to you? Yeah no, it's probably not.
Right now is the time where we put on our thinking hats and deliberate on "What do we do if we can't compute recommendations based on orders?"

We could:

- Find a recommendation that we can always make (most popular products, anyone?)

- Return an error to the frontend servers and tell them deal to with it. That's a perfectly valid strategy, maybe it's not even a big deal to not present recommendations on the home page.

Both strategies can be applied using a Circuit Breaker. The pattern is agnostic of how you deal with the detected failure. It allows you to not overload the external system further. Furthermore, you can **respond quicker** instead of running into timeouts over and over, when it is highly probable that it will happen again (this is known as the *Fail Fast* principle). This way, propagation of failure and slowness through the dependency chain is mitigated. Your frontend servers will go back to responding quickly after the recommendation system's Circuit Breaker kicks in after a few query timeouts.


## Conclusion
Circuit Breakers are a very useful pattern to deal with bursts of failure, preserving quick response times and facilitating a consciously chosen fallback strategy (whatever it might look like).
There are a lot of examples from the practice where the same principle is applied (e.g., NGINX upstreams and Kubernetes services), but that is for another time. Until next time!
