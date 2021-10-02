---
layout: post
title:  "How does a Load Balancer scale?"
categories: Cloud, Distributed Systems, How  Stuff Works
---

A Load Balancer helps scale your application, but how does the Load Balancer scale? How comes it does not become the single point of failure for your application as load increases?

I mean: we expect a load balancer endpoint to resolve to one IP. We know an IP is associated with a single piece of hardware. 

Is it that load balancers are backed by powerfull single hardware devices that can never fail? Don't think so. 

 So, I am interested in how AWS solves this problem. I will look specificaly at the AWS Network Load Balancer (NLB). An NLB is advertised to handle TK millions of requests per second. How can it do that?




