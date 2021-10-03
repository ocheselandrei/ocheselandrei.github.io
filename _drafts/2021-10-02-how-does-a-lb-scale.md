---
layout: post
title:  "How does a Load Balancer scale?"
categories: Cloud, Distributed Systems, How  Stuff Works
---

A Load Balancer helps scale your application, but how does the Load Balancer scale? How comes it does not become the single point of failure for your application as load increases?

If the Load Balancer endpoint resolves to a single IP, then it must have a single piece of hardware associated with that IP. What if that single piece of hardware fails? Single point of failure.

If it does scale out horizontally and resolves to multiple node IPs, then how does it transparently spread the load between those IPs? It can't be DNS, right? DNS is slow to react to mapping changes, a lot of caching layers are involved, and clients may get stuck calling the same IP and thus the same node, and thus single point of failure. While with Load Balancers, you just expect to place them in front of your application instances and your single point of failure problem is resolved.

So, let's see how AWS solves this Load Balancer scaling problem. I will look specificaly at the AWS Network Load Balancer (NLB). An NLB is advertised to handle TK millions of requests per second. How can it do that?

Right from the start, it is clear that NLB uses multiple internal nodes, so it does scale horizontally.

Will zoom in on what happens in a single Availabily Zone (AZ).

The documentation says it places a network interface (NI) in that AZ, and the nodes use a static IP provided by that NI. 

Wow. So multiple hardware instances may share the same IP. But then, who does the load balancing between those instances? How is traffic split between them? But no, none of that magic. Let's move on.

Further documentation clarifies things. The NLB will add more nodes in an AZ as traffic increases, and all those nodes will be placed behind a DNS hostname. 

**So YES, it does delegate to DNS for scaling**.

And so Yes, you do need to worry about how clients resolve the NLB internal IPs, just like you would need to worry if clients would use any DNS hostname.

But if NLB places nodes behind a DNS hostname, why wouldn't you just place your application instances directly behind that DNS? You would skip an extra network hop in the process and thus decrease the latency.

Well, you could, but then you would have to replicate the DNS registration work that the NLB is doing. You would need to constantly update the DNS records as your application instances come and go. And you probably have better things to do. But it is possible, and other AWS services are actually doing it.

One example is Amazon OpenSearch, which does this extra work of placing all the data node instances directly behind a DNS hostname, with no extra Load Balancer involved. At first glance, this may seem worrisome, what if you end-up overloading a single data node, why don't they provide a Load Balancer instead that just takes all these problems away?

Well, what they do is actually OK. If they provided an NLB, DNS would still be involved, and the same problems would be present, only shifted to a different layer.

So how this DNS mapping work and why don't you generally have to worry about it?

The DNS mapping advertises an expiration interval.






