---
layout: post
title:  "A journey into understanding how load balancers scale"
categories: Cloud Architecture, Distributed Systems, How  Stuff Works
---
## What is this about?

We all know that a Load Balancer helps horizontally scale an application, but how does the Load Balancer itself scale?

How come it does not become itself the Single Point of Failure (SPOF) for the application?

Thus, I started a journey towards understanding how Load Balancers scale under the hood, with an emphasis on Cloud environments, where horizontal scaling is the norm.

## Initial AWS documentation research

I started researching the existing AWS documentation about the Elastic Load Balancing service.

For the Application Load Balancer (ALB), which operates at Layer 7 (Application), it was easy to understand. The ALB scales out its internal nodes and then delegates to DNS to map its endpoint name to all the internal node IPs. **The client is then responsible to properly handle DNS resolution**. There is a lot of documentation on how to deal with DNS resolution on the client, TK, so not going to insist here.

But the Network Load Balancer (NLB), which operates at Layer 4 (Transport) was challenging to understand. Based on initial AWS documentation I read, I thought the scaling process still relies on DNS, just like for the ALB, and this was going to be a very short article. But then, several blog posts pointed out that the NLB provides a **single static IP** per Availability Zone (AZ), which can be directly used in customer firewalls, so DNS can be taken out of the scaling equation.

## The question

If a Load Balancer resolves to a single static IP, then how doesn't it become a SPOF?

I knew that one IP is associated with a single device on a network.

That device could be a strong piece of hardware that scales vertically and never fails. It may run in a primary-standby setup for the never fails part. But vertical scaling is not something I expect to happen and be practical in a cloud environment.

But if a Load Balancer in the Cloud does scale horizontally, then it will have multiple internal nodes.

How can multiple nodes (instances/devices) share the same IP and get an equal share of the traffic to that IP? It's like a Load Balancer on top of a Load Balancer...

This is the question the following research will focus on.

## The research

I started reading blogs about load balancing and learned interesting things about load balancing history, with DNS being the initial form of scaling. TK

I already saw in that article a diagram that says yes, multiple devices listen to the same IP, but not clear how that actually happens. TK

After reading more articles, I still didn't know how multiple devices can share the same IP.

So I started thinking of a related problem that AWS solves: NAT Gateways. NAT Gateways translate several private IPs to a single public IP. NAT Gateways handle any traffic that you throw at them, so they must horizontally scale somehow. But I did not find any relevant documentation on this either.

So then I started thinking about the poor relative of the NAT Gateways, which is NAT instances. I'm sure someone had to come up with a way to horizontally scale NAT instances. I found one article TK about multiple NAT instances in AWS. The article mentioned a way of moving the same Elastic Network Interface (network card equivalent, provides IP) from one primary NAT instance (EC2 instance) to a failover one, if the primary instance failed. This did not help much. But the article also mentioned implementing a variant of Anycast protocol. Now this protocol name sounded promising, I've seen it before in some Quora searches, so I started researching it.

I eventually found the Wikipedia article on Anycast. And light started to shine.

> **Anycast** is a network [addressing](https://en.wikipedia.org/wiki/Addressing "Addressing") and [routing](https://en.wikipedia.org/wiki/Routing "Routing") methodology in which a single destination [IP address](https://en.wikipedia.org/wiki/IP_address "IP address") is shared by devices (generally servers) in multiple locations.

While following links related to Anycast, I found a Wikipedia page (and I really can't find it again at the time of writing) that pointed me to 2 reference implementations of large scale network load balancers: the AWS Hyperplane and the Google Maglev. So I started studying those.

### AWS Hyperplane

Some AWS Hyperplane documentation is available as re:Invent or other conference videos. I looked at these: [networking-scale-2018-load-balancing-at-hyperscale](https://atscaleconference.com/videos/networking-scale-2018-load-balancing-at-hyperscale/) and [AWS re:Invent 2017: Another Day, Another Billion Flows (NET405)](https://www.youtube.com/watch?v=8gc2DgBqo9U&t=2066s). I wish the service had a paper as well.

Several interesting things and insights I got from there:

* NLB and NAT Gateways are indeed similar problems that AWS solves with a single private service, the AWS Hyperplane. The AWS Hyperplane provides a shared fleet with cell-based architecture, that backs implementation for: NLB, NAT Gateways, Private Links, and more.
* When you create an NLB, there are no dedicated instances started especially for you, like it would happen if you start an Elastic Map Reduce or Amazon OpenSearch (former AWS Elasticsearch) cluster. You share existing AWS Hyperplane nodes (EC2 instances) with other customers, but AWS ensures that the NLB acts as if it is isolated and dedicated to you.
* There is cool engineering involved in ensuring proper TCP connection tracking across multiple Load Balancer nodes. It solves an in-memory distributed consensus problem across all customer defined NLBs with million TPS possible per NLB. Nice.

But AWS Hyperplane videos still did not provide details on how multiple Load Balancer instances share the same IP, so I moved on to the Google Maglev paper.

### Google Maglev

Google Maglev is Google's software Network Load Balancer implementation. The paper clarifies the technologies used to put several instances behind the same IP. Insights I got from this paper:

* Maglev load balancer nodes rely on network routers to distribute traffic evenly to each load balancer node.
* The network routers run the Border Gateway Protocol (BGP) with Equal Cost Multipath (ECMP) strategy. Every load balancer node advertises itself as a next hop in reaching a load balancer static IP, with equal cost between the nodes. The routers then shuffle the traffic destined to the Load Balancer IP across all the corresponding nodes.
* Maglev calls the load balancer static IP a Virtual IP (VIP). A VIP DOES NOT CORRESPOND TO A PHYSICAL IP ADDRESS ON THE NETWORK. That's the magic that makes it possible to have multiple nodes share the same IP. The load balancer nodes will tell the routers: if someone sends traffic to this static IP (the VIP), then redirect that traffic to my physical IP, like I'm a mini-router, and I'll know how to deal with it.

But the paper doesn't spend too much time on the BGP + ECMP configuration details, and I was interested to learn more about how this works specifically, so I moved on to yet another Network Load Balancer implementation, the Github Load Balancer.

### Github Network Load Balancer

The Github Load Balancer also uses BGP + ECMP. Some extra insights apart from Google Maglev:

* Their documentation has a dedicated chapter and gives a name to the specific question I had from the begining of this research: [Stretching an IP](https://github.blog/2016-09-22-introducing-glb/#stretching-an-ip)
* They make readers (me included) not feel bad about not already knowing these networking details related to "stretching" an IP.

> Typically we think of an IP address as referencing a single physical machine, and routers as moving a packet to the next closest router to that machine. [...] In reality, most networks are far more complicated.

But they also skip on BGP + ECMP configuration details (like examples of the internal paths database - the RIB, and the route table entries). And I reached the conclusion they are right to do so. BGP is a complex protocol. After several hours trying to make sense of BGP configuration examples from Internet articles, I concluded it is a waste of time, unless you are a network engineer who needs to deal with this on a day by day basis. Otherwise these things will dissapear from your memory anyway. Knowing it is possible is good enough.

Yet, out of all those BGP related articles I looked at, I found the Wikipedia entry on BGP the most helpful in providing that little extra detail on how an architecture based on BGP + ECMP would look like for implementing a horizontally scalable Network Load Balancer.

### BGP + ECMP architecture for horizontally scalable Network Load Balancers

The Wikipedia entry on BGP provides this helpful architecture diagram on clients, Internal BGP (iBGP) routers and External BGP (eBGP) routers interaction, which can be easily applied to Network Load Balancer implementation as well:

![iBGP and eBGP](https://upload.wikimedia.org/wikipedia/commons/thumb/6/6e/RR_BGP.svg/300px-RR_BGP.svg.png)

How does this map to a Network Load Balancer architecture:

* Clients: these map to the Network Load Balancer nodes. **They are BGP routers themselves in order to talk to other BGP routers**. They advertise themselves as equal cost next hops in the RR paths towards backend services. The backend services are identified by a Virtual IP, which corresponds to the static IP provided by a Network Load Balancer created in a Cloud, such as the AWS Network Load Balancer. The BGP protocol allows these client iBGP routers to not exchange routes between them, which saves network bandwith and processing power.
* RR: Route Reflectors. These map to the BGP routers mentioned in the Google Maglev and Github Load Balancer architectures. They are iBGP routers that collect routes from load balancer nodes and synchronize them with peer iBGP RRs and eBGP routers.
* EBGP: eBGP router that advertises public facing (public IP) Network Load Balancers.

Questions I had:

* If I delegate load balancing to a network router, doesn't that network router become a SPOF? Well, as seen from the diagram, BGP allows route synchronization (replication) between peer routers. So SPOF is mitigated by replication.
* Why use such a complex protocol, designed for advertising routes to public IP prefixes between Autonomous Systems (Internet Service Providers, large companies, organizations, etc.) on the Internet, for Network Load Balancer limited purposes, which many times does not even have to expose a public IP to the world? Well, again, in my opinion, the scalability qualities of BGP (route replication) make it suitable for the horizontal scaling requirements of a Network Load Balancer.

## Takeaway

Load Balancers scale by delegating to other systems.

Some load balancers horizontally scale by delegating to DNS (the AWS ALB example).

Network Load Balancers scale by delegating to network routers running Anycast type protocols. The actual protocols that look to be used in industry are BGP with ECMP.

You cannot implement a Network Load Balancer at the level of abstraction of a distributed system: nodes and messages. You can't just start 3 EC2 instances, have them send some messages between them, and get a horizontally scalable Network Load Balancer. You would need to go talk to the Networking team, and start diving into a rabbit hole of networking protocols details.

Thus, the Network Load Balancer is truly a distributed systems architecture building block: easy to consume for a distributed system implementation, but hard to implement.
