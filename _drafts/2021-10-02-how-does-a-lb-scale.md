---
layout: post
title:  "How does a Load Balancer scale?"
categories: Cloud, Distributed Systems, How  Stuff Works
---

A Load Balancer helps scale your application, but how does the Load Balancer scale? How comes it does not become the single point of failure for your application as load increases?

If the Load Balancer endpoint resolves to a single IP, then it must have a single piece of hardware associated with that IP. What if that single piece of hardware fails? Single point of failure.

If it does scale out horizontally and resolve to multiple Load Balancer node IPs, then how does it transparently spread the load between those internal IPs? It can't be DNS, right? DNS is slow to react to mapping changes, a lot of caching layers are involved, and clients may get stuck calling the same IP and thus the same node, and thus single point of failure. While with Load Balancers, you just expect to place them in front of your application instances and your single point of failure problem is gone.

So, let's see how AWS solves this Load Balancer scaling problem. I will look specificaly at the AWS Network Load Balancer (NLB). 

Will start with an NLB specific doc, [What is a Network Load Balancer?](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html)

An NLB is said to handle millions of requests per second. How can it do that?

Right from the start, it is clear that NLB uses multiple internal nodes, so it does scale horizontally:

>When you enable an Availability Zone for the load balancer, Elastic Load Balancing creates a load balancer node in the Availability Zone.

>Elastic Load Balancing creates a network interface for each Availability Zone you enable. Each load balancer node in the Availability Zone uses this network interface to get a static IP address.

Will zoom in on what happens in a single Availabily Zone (AZ).

As seen above, the documentation seem to say that NLB places a network interface (NI) in that AZ, and the nodes share a static IP provided by that NI. 

Wow. So multiple hardware devices could share the same IP. But then, who does the load balancing between those devices?  

Lets look into further documentation, [How Elastic Load Balancing works](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html)

This clarifies things. No multiple nodes per single IP magic. The NLB will add more nodes in an AZ as traffic increases, and all those nodes will be placed behind the NLB DNS hostname.

>The Amazon DNS servers return **one or more IP addresses** to the client. These are the IP addresses of the load balancer nodes for your load balancer.

>As traffic to your application changes over time, Elastic Load Balancing scales your load balancer and updates the DNS entry.

**So YES, it does delegate to DNS for scaling**.

And finally, the icing on the cake:

>**The client determines which IP address to use** to send requests to the load balancer.

And so Yes, you do need to worry about how clients resolve the NLB internal IPs, just like you would need to worry if clients would use any other round-robin DNS hostname.

But if NLB places nodes behind a DNS hostname, why wouldn't you just place your application instances directly behind a DNS yourself? You would skip an extra network hop in the process and thus decrease the latency.

Well, you could, but then you would have to replicate the DNS registration work that the NLB is doing. You would need to constantly update the DNS records as your application instances come and go. You would need to handle TLS termination and certificate management. And you probably have better things to do. But it is possible, and other AWS services are actually doing it.

One example is Amazon OpenSearch (former AWS Elasticsearch), which does this extra work of placing all the data nodes directly behind a DNS hostname, with no extra Load Balancer involved. At first glance, this may seem worrisome, what if you end-up overloading a single data node, why don't they provide a Load Balancer instead that just takes all these problems away?

Well, as we've seen, what they do is actually OK. If they provided an NLB, DNS would still be involved, and the same DNS problems would be present, only shifted to a different layer.

## So what should your Load Balancer client do?

The DNS hostname will do several things for you: resolve to the list of all the NLB node IPs; shuffle that list so that clients who look only at the first IP still use different IPs between different processes; send a header to ask you to refresh that list every   0 seconds. 

But from there, the client needs to take over and respect what the DNS is asking it to do, or even over-do it:  0 seconds refresh interval may still be too long if traffic spikes instantly.






