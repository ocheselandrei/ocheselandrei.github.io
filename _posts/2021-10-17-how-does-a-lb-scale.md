---
layout: post
title:  "A Journey Into Understanding How Load Balancers Scale"
categories: Cloud Architecture, Distributed Systems, How Stuff Works
---
## What is this about?

We all know that a Load Balancer helps horizontally scale an application, but how does the Load Balancer itself scale?

How come it does not become itself the Single Point of Failure (SPOF) for the application?

Thus, I started a journey towards understanding how Load Balancers scale under the hood, with an emphasis on Cloud environments, where horizontal scaling is the norm.

## The initial questions

I started researching the existing AWS documentation about the AWS Elastic Load Balancing Service (ELB), with a focus on the Network Load Balancer (NLB) it provides, as it is advertised to handle millions TPS.

After reading [What is a Network Load Balancer?](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html#network-load-balancer-overview), I was confused.

> Elastic Load Balancing creates a network interface for each Availability Zone you enable. Each load balancer node in the Availability Zone uses this network interface to get a static IP address.

"Each load balancer node in the Availability Zone" sounds like the NLB can have multiple internal nodes per AZ, thus it would scale horizontally. But then, each node "uses this network interface to get a static IP address". How can that be? How can a single network interface (ENI, network card equivalent) provide IPs for multiple nodes? And more, how can it provide the same static IP to multiple nodes? How can multiple nodes on a network share the same IP? Lots of questions.

After reading [How Elastic Load Balancing works?](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html), I was still confused.

It looks like DNS is involved to publish a list of horizontally scaled internal nodes to clients:

> The Amazon DNS servers return one or more IP addresses to the client. **These are the IP addresses of the load balancer nodes** for your load balancer.

> As traffic to your application changes over time, **Elastic Load Balancing scales your load balancer and updates the DNS entry**.

> **The client determines which IP address to use** to send requests to the load balancer.

Ignoring my confusion, having the NLB horizontally scale its internal nodes and then registering them with DNS and letting clients choose the IP seemed like a plausible explanation for how it scales. And this was about to be a very short and not very interesting article.

But then the blog post [New Network Load Balancer – Effortless Scaling to Millions of Requests per Second](https://aws.amazon.com/blogs/aws/new-network-load-balancer-effortless-scaling-to-millions-of-requests-per-second/) made clear several aspects:

> **Static IP Addresses** – Each Network Load Balancer provides **a single IP address** for each Availability Zone in its purview

So surely the NLB does not provide multiple IPs per AZ.

> Network Load Balancer can be used in situations where IP addresses need to be hard-coded into DNS records, customer firewall rules, and so forth.

**So for NLB, DNS can be taken out of the scaling equation.** NLB provides a single IP (per AZ) and that IP can be hardcoded in firewall rules, so it better not change.

And then, if the NLB advertizes a single, never-changing IP to customers, how can it put multiple internal nodes behind that IP, in order for it to horizontally scale to millions TPS? Assuming of course (and hoping) that it does scale horizontally.

These are the questions that motivated the rest of this research into Load Balancer scaling.

## The research

I started reading blogs about Load Balancing in general and learned interesting things about Load Balancing history, with DNS being the initial form of scaling.

For example this article: [What is Load Balancing?](https://devcentral.f5.com/s/articles/what-is-load-balancing-24740). I already saw in that article a diagram that says yes, multiple devices can listen to the same IP (192.0.2.1 in that diagram), but still not clear how that can actually happen:

![2 devices same IP](https://devcentral.f5.com/servlet/rtaImage?eid=ka01T000000HtSm&feoid=00N1T00000At9R1&refid=0EM1T000001Mr9x)

After reading more articles, I still didn't know how multiple devices can share the same IP.

So I started thinking of a related problem that AWS solves: NAT Gateways. NAT Gateways translate several private IPs to a single public IP. NAT Gateways handle any traffic that you throw at them, so they must horizontally scale somehow. But I did not find any relevant documentation on this either.

So then I started thinking about the poor relative of the NAT Gateways, which is NAT instances. I'm sure someone had to come up with a way to horizontally scale NAT instances. I found one Github repository: [AWSnycast](https://github.com/bobtfish/AWSnycast) about high-availability/failover with multiple NAT instances in AWS. I did not understand much of the documentation when I first read it, but it mentioned implementing a variant of Anycast protocol. Now this protocol name sounded promising, I've seen it before in some of my previous Quora searches, so I started researching it.

I eventually got to the Wikipedia article on [Anycast](https://en.wikipedia.org/wiki/Anycast). And light started to shine.

> **Anycast** is a network [addressing](https://en.wikipedia.org/wiki/Addressing "Addressing") and [routing](https://en.wikipedia.org/wiki/Routing "Routing") methodology in which a single destination [IP address](https://en.wikipedia.org/wiki/IP_address "IP address") is shared by devices (generally servers) in multiple locations.

While following links related to Anycast, I found a Wikipedia page (and I really can't find it again at the time of writing) that pointed me to 2 reference implementations of large scale Network Load Balancers: the AWS Hyperplane and the Google Maglev. And light shined even more.

### AWS Hyperplane

Some AWS Hyperplane documentation is available as re:Invent or other conference videos. I looked at these: [networking-scale-2018-load-balancing-at-hyperscale](https://atscaleconference.com/videos/networking-scale-2018-load-balancing-at-hyperscale/) and [AWS re:Invent 2017: Another Day, Another Billion Flows (NET405)](https://www.youtube.com/watch?v=8gc2DgBqo9U&t=2066s). I wish the service had a paper as well.

Several interesting things and insights I got from there:

* NLB and NAT Gateways are indeed similar problems that AWS solves with a single private service, the AWS Hyperplane. The AWS Hyperplane provides a shared fleet with cell-based architecture, that backs implementation for: NLB, NAT Gateways, Private Links, and more.
* When you create an NLB, there are no dedicated instances started especially for you, like it would happen if you start an Elastic Map Reduce or Amazon OpenSearch (former AWS Elasticsearch) cluster. You share existing AWS Hyperplane nodes (EC2 instances) with other customers, but AWS ensures that the NLB acts as if it is isolated and dedicated to you.
* There is cool engineering involved in ensuring proper TCP connection tracking across multiple Load Balancer nodes. It solves an in-memory distributed consensus problem across all customer defined NLBs with million TPS possible per NLB. Nice.

But AWS Hyperplane videos still did not provide details on how multiple Load Balancer instances share the same IP, so I moved on to the Google Maglev paper.

### Google Maglev

Google Maglev is Google's software Network Load Balancer implementation. The [paper](https://research.google/pubs/pub44824/) clarifies the technologies used to put several instances behind the same IP. Insights I got from this paper:

* Maglev Load Balancer nodes rely on **network routers to distribute traffic evenly** to each load balancer node.
* The network routers run the **Border Gateway Protocol (BGP)** with **Equal Cost Multipath (ECMP)** strategy. Every load balancer node advertises itself as a next hop in reaching the equivalent of a NLB static IP, with equal cost between the nodes. The router then shuffles the traffic across the load balancer nodes.
* Maglev calls the equivalent of the NLB static IP a Virtual IP (VIP). A VIP DOES NOT CORRESPOND TO A PHYSICAL IP ADDRESS ON THE NETWORK. That's the magic that makes it possible to have multiple nodes share the same IP. The load balancer nodes will tell the routers: if someone sends traffic to this static IP (the VIP), then redirect that traffic to my physical IP, like I'm a mini-router, and I'll know how to deal with it.

But the paper doesn't spend too much time on the BGP + ECMP configuration details, and I was interested to learn more about how this works specifically, so I moved on to yet another Network Load Balancer implementation, the Github Load Balancer.

### Github Network Load Balancer

The [Github Load Balancer](https://github.blog/2016-09-22-introducing-glb/) also uses BGP + ECMP. Some extra insights apart from Google Maglev:

* Their documentation has a dedicated chapter and gives a name to the specific question I had from the begining of this research: [Stretching an IP](https://github.blog/2016-09-22-introducing-glb/#stretching-an-ip)
* They make readers (me included) not feel bad about not already knowing these networking details related to "stretching" an IP.

> Typically we think of an IP address as referencing a single physical machine, and routers as moving a packet to the next closest router to that machine. [...] In reality, most networks are far more complicated.

But they also skip on BGP + ECMP configuration details (like examples of the internal paths database - the RIB, and the route table entries). And I reached the conclusion they are right to do so. BGP is a complex protocol. After several hours trying to make sense of BGP configuration examples from Internet articles, I concluded it is a waste of time, unless you are a network engineer who needs to deal with this on a day by day basis. Otherwise these things will dissapear from your memory anyway. Knowing what is possible is good enough.

Yet, out of all those BGP related articles I looked at, I found the Wikipedia entry on [BGP](https://en.wikipedia.org/wiki/Border_Gateway_Protocol) the most helpful in providing that little extra detail on how an architecture based on BGP + ECMP would look like for implementing a horizontally scalable Network Load Balancer.

### BGP + ECMP architecture for horizontally scalable Network Load Balancers

The Wikipedia entry on [BGP](https://en.wikipedia.org/wiki/Border_Gateway_Protocol) provides this helpful architecture diagram on clients, Internal BGP (iBGP) routers and External BGP (eBGP) routers interaction, which can be easily applied to Network Load Balancer implementation:

![iBGP and eBGP](https://upload.wikimedia.org/wikipedia/commons/thumb/6/6e/RR_BGP.svg/300px-RR_BGP.svg.png)

How does this map to a Network Load Balancer architecture:

* Clients in the diagram: these map to the Network Load Balancer nodes. **They are BGP routers themselves in order to talk to other BGP routers**. They advertise themselves as equal cost next hops in the RR paths towards backend services. The backend services are identified by a Virtual IP, which corresponds to the static IP provided by a Network Load Balancer created in a Cloud, such as the AWS Network Load Balancer. The BGP protocol allows these client iBGP routers to not exchange routes between them, which saves network bandwith and processing power.
* RR in the diagram: [Route Reflectors](https://en.wikipedia.org/wiki/Border_Gateway_Protocol#Route_reflectors). These map to the BGP routers mentioned in the Google Maglev and Github Load Balancer architectures. They are iBGP routers that collect routes from load balancer nodes and synchronize them with peer iBGP RRs and eBGP routers.
* EBGP: eBGP router that can advertise public facing (public IP) Network Load Balancers.

Questions I had:

* If I delegate load balancing to a network router, doesn't that network router become a SPOF? Well, as seen from the diagram, BGP allows route synchronization (replication) between peer routers. So SPOF is mitigated by replication.
* Why use such a complex protocol, designed for advertising routes to public IP prefixes between Autonomous Systems (Internet Service Providers, large companies, organizations, etc.) on the Internet, for Network Load Balancer limited purposes, which many times does not even have to expose a public IP to the world? Well, again, in my opinion, the scalability qualities of BGP (route replication) make it suitable for the horizontal scaling requirements of a Network Load Balancer.

## Takeaway

Network Load Balancers scale by delegating to network routers running Anycast type protocols. The actual protocols that look to be used in industry are BGP with ECMP.

You cannot implement a Network Load Balancer at the level of abstraction of a distributed system: nodes and messages. You can't just start a couple EC2 instances, have them send some messages between them, and get a horizontally scalable Network Load Balancer. You would need to go talk to the Networking team, and start diving into a rabbit hole of networking protocols details.

Thus, the Network Load Balancer is truly a distributed systems architecture building block: easy to consume for a distributed system implementation, but hard to implement and having to go to lower abstraction levels for implementation.

As a next step from here, it is fascinating to further see and understand how the 3 different Network Load Balancer implementations mentioned in this article use 3 different solutions for the same hard problem: distributed TCP flow tracking at scale.
