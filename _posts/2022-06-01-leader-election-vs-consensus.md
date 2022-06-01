---
layout: post
title:  "Leader Election vs Distributed Consensus - Which One Is Harder?"
tags: [Distributed Systems, DSLabs]
---

### Introduction

Long time ago, when I interviewed at Amazon, I was asked about distributed Leader Election. Few years after, when I became an interviewer, I found out Leader Election was one criterion, at the time, to differentiate senior candidates.  

In contrast, Paxos, a distributed Consensus algorithm, was only briefly mentioned at the end of one presentation, delivered by the principal engineer in our office, who looked like the only one able (and willing) to understand those things.

So it always seemed to me that Leader Election is the practical, simpler problem that an engineer would be expected to reasonably implement, while the Consensus is the abstract, complex problem that is better left to an existing implementation.

When I needed to deal with Leader Election related problems (like once), I just used Zookeeper recipes, and did not worry too much about the theory. 

But recently, I started working on [DSLabs](https://github.com/emichael/dslabs) as a "hobby", and made it to the Paxos lab. And then, along the line of "Can Batman beat Superman?", the question popped back into my mind:

**Is Leader Election actually easier than Consensus?**

On one hand, the well-known Consensus algorithms (Paxos, Raft) use Leader Election as a building block. Paxos doesn't even specify which Leader Election algorithm to use, like it's too trivial to bother with. So Leader Election does seem simpler than Consensus in theory.

On the other hand, if you can solve Leader Election, then you can solve Consensus, because all the distributed system nodes can agree on using the leader value. And all the nodes must reach consensus on which is the Leader.

So it looked to me like Leader Election and Consensus are actually equivalent, and we run into a chicken and egg problem. To solve Consensus, you need Leader Election, which requires Consensus. I had to dive deeper on this.

### The Consensus problem 
A set of nodes propose a set of values. All nodes must choose the same value out of the set of proposed values. 

For more details, I recommend [Paxos Made Simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf), which is an example in what precise, clear, and simple writing means.

#### Why Consensus?
Consensus becomes most useful when it happens over multiple rounds of picking a single value, to obtain a global, totally ordered sequence of values. 
Then you can implement State Machine Replication, where values are commands to execute. 
Then you can implement fault-tolerant, single-copy consistency in all kinds of stateful distributed systems, like databases. 

Being able to maintain consistent, replicated state across multiple machines always seemed cool to me.

### The Leader Election problem
A special case of Consensus, where the set of proposed values is the set of nodes themselves. 
All nodes will agree on a single node to be the leader.

#### Why Leader Election?
The leader is the single node with the power to take a certain action: run a job, write to a database, make a service call, etc.
For some systems, this is important to preserve correctness, or protect from putting too much load on resources.

The single leader can provide that global totally ordered sequence of commands needed for State Machine Replication, just like Consensus does.
But we will clarify this relationship with Consensus in the rest of this article.

### The online search answer

I started of-course by searching online whether Leader Election and Consensus are equivalent problems, and found a useful answer: [What is the difference between Consensus and Leader Election problems?](https://cs.stackexchange.com/questions/105716/what-is-the-difference-between-consensus-and-leader-election-problems)

That answer helped me understand the following:
* As a special case of Consensus, Leader Election has a simpler part. Assuming the nodes know about all the other nodes in the system (the "configuration"), they can just choose locally the one with the lowest "id" (IP, DNS name). This is what Viewstamped Replication is doing. 
* But it also has a more complicated part, an extra requirement. When the leader fails, another process must become the **single** leader. 
* [Scientific research](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.51.1256&rep=rep1&type=pdf) provides a definitive answer:
> In the presence of failures, Leader Election is harder than Consensus. There are failures modes under which you can solve Consensus, but not Leader Election. 

This was totally counter-intuitive to me. Paxos and Raft Consensus algorithms use Leader Election as a building block. So they should be as hard as the Leader Election. 

Are there other theoretical Consensus algorithms that don't need Leader Election, but nobody uses in real-world?

And what do those failure modes mean? Under what failure modes does Consensus work, but Leader Election doesn't?

So moved on to researching a trail of scientific papers starting from the one mentioned in the online answer.

### The scientific answer details

The [FLP impossibility](https://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf) result says you can't solve Consensus in an asynchronous distributed system when at least one process may fail.

An asynchronous distributed system is one in which there is no bound on message delay, clock drift, or time necessary to execute a step. 
This is the model that best resembles the real-world. 
Even in a cloud datacenter, with high performance networks and time synchronization services, you can't expect a system to be synchronous all the time. 

The Consensus impossibility comes from the difficulty of knowing whether a process crashed (failed) or is very slow. 
This difficulty can be encapsulated inside a module called an Unreliable Failure Detector. 

This Unreliable Failure Detector module encapsulates the non-deterministic part of a Consensus algorithm. 
By introducing non-determinism (e.g. random heartbeat timeouts), the FLP impossibility can be circumvented and Consensus can be solved. 

An Unreliable Failure Detector module can make mistakes about whether a process is failed or not, and it has several classifications depending on the type of mistakes it can make.
So the failure models mentioned in the online answer actually refer to the different classifications of this module.

The paper [Unreliable Failure Detectors for Reliable Distributed Systems](https://www.cs.utexas.edu/~lorenzo/corsi/cs380d/papers/p225-chandra.pdf) proves that Consensus can be solved with a Failure Detector that provides Weak Accuracy: it may suspect a correct process to have failed, but eventually will stop suspecting it. A correct process can appear as failed for multiple reasons: it is stuck in Garbage Collection, the network is slow or drops messages. We can't expect to do better than this in the real-world. 

The paper [Election Vs. Consensus in Asynchronous Systems](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.51.1256&rep=rep1&type=pdf) proves that Leader Detection cannot be solved with a Failure Detector that provides Weak Accuracy. There is a lot of math in there, but the proof if simple. Say you have 2 processes, i and j, i is the leader. i becomes slow, j suspects it as failed. j has no chance but to become the leader, because the algorithm requires a leader at all times. But i is not actually failed, and still knows it is the leader. Thus, you cannot satisfy the requirement of **single** leader.

### The theoretical light

So yes, now I understand why Leader Detection is harder, but still I don't understand how Paxos and Raft can use it and still require less failure detection perfection to solve?

The answer came from yet another paper, somehow unrelated, but most useful in clarifying my questions: [Distributed Eventual Leader Election
in the Crash-Recovery and General Omission Failure Models](https://addi.ehu.es/bitstream/handle/10810/45410/TESIS_FERNANDEZ_CAMPUSANO_CHRISTIAN.pdf?sequence=1).

I realized that what Paxos and Raft are using is called **Eventual Leader Election**, an algorithm that eventually converges to answering with the same leader across all the system nodes. It will answer with a leader id for each process in the system, but that leader id may be different across the processes for a while.

From the Paxos and Raft specifications, it was clear that they don't need a single leader guarantee. Paxos stays correct even without electing a leader, or with multiple leaders, but will most probably not make progress until a single leader is chosen. Raft also stays correct across periods of split votes, when there is no leader. I just never heard before that this is called Eventual Leader Election, as a separate problem from Leader Election.

Eventual Leader Election is simpler than Consensus, because it does not have to be perfect, like the Leader Election problem requires. It can make mistakes, it can choose multiple leaders, and the Consensus on-top can work around those mistakes.

In a real-world asynchronous distributed system, it is impossible to solve Leader Election, because the single leader requirement involves a Perfect Failure Detector, one that never mistakes a correct process as failed. And this perfection is not possible.

So the final answer is:

**There is no Leader Election in real world distributed systems.**

**There is only Eventual Leader Election. You must deal with multiple leaders at one time. And yes, this one is simpler than Consensus.**

### What does this mean in practice?

So how will you deal with Leader Election, which is such a hard and impossible problem, in practice?

A very good answer comes from the AWS documentation: [Leader Election in Distributed Systems](https://aws.amazon.com/builders-library/leader-election-in-distributed-systems/).

First and foremost, the document states that Leader Election that guarantees a single leader at all times is impossible in distributed systems (referring to real-world, asynchronous system models). 
> In distributed systems, it’s not possible to guarantee that there is exactly one leader in the system.

The above is buried in the middle of the document, but should have been the first sentence. Otherwise, such an important sentence is easy to miss and subtle bugs can be introduced.

So you can only hope to implement Eventual Leader Election, where you will have to deal with 0 or 2 leaders at a time. If you need the correctness of a single leader operating at a time, you need to put some extra distributed glue on top. Consensus, Primary-Backup algorithms are such algorithms that already put the extra glue. They "trust but verify". The node who thinks it is leader validates this assumption at each operation by coordinating with other nodes.

The paper mentions two existing, proven implementations to re-use in your systems instead of implementing your own distributed algorithms from scratch: [Apache ZooKeeper](https://zookeeper.apache.org/doc/current/recipes.html) and [DynamoDb Lock Client](https://github.com/awslabs/dynamodb-lock-client).

ZooKeeper implements a Consensus algorithm ([ZAB](https://zookeeper.apache.org/doc/r3.4.13/zookeeperInternals.html)) on top of Eventual Leader Election. It also provides recipes to its users to implement Leader Election, or Distributed Locking, which are tightly related problems. Similar recipes are provided by the DynamoDB Lock Client.

So in practice you can implement Eventual Leader Election by delegating to an external Consensus service which runs on top of an Eventual Leader Election algorithm. 

But if I delegate to a Consensus-providing service to maintain my system's leader, and Consensus can guarantee correct agreement on a single value, doesn't this mean I get guaranteed single-leader Leader Election?

Well, no, the Leader Election impossibility in real world still holds. The external Consensus system will know about a single leader at a time, but 2 of your system nodes will still believe they are both leaders. 

The node who was leader may encounter a long garbage collection pause. Another node will become leader. After the garbage collection pause ends, the old leader will still think it is leader. They will both try to write to your shared storage. The leaders need to receive a "fencing token" (auto-incrementing number) from the Consensus service, and **your shared storage must cooperate** by implementing atomic conditional write operations that validate that the fencing token never goes back.

A must-read here, with a lot better explanation of the above issues in the context of Distributed Locking, is this blog article: [How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html), by Martin Kleppmann, author of [Designing Data-Intensive Applications](http://dataintensive.net/), another must-read book.

As a closing note, if your shared storage can cooperate by providing atomic conditional writes, do you actually need an extra system for Leader Election? You may need it to reduce the overhead of write operations, but it is something to consider to simplify your system.

### Takeaway

There is only Eventual Leader Election in real-world distributed systems. Consensus algorithms stay correct in the face of Eventual Leader Election, and only Eventual Leader Election can be implemented on top of Consensus algorithms. 

If you build your own distributed algorithm on top of Eventual Leader Election, you need to ensure correctness with 0 or 2 leaders at the same time.

But there are already distributed algorithms that are correct in the face of imperfect Leader Election: Consensus (Paxos, Raft, Viewstamped Replication), Primary-Backup. Consider using these algorithms instead, as provided by an existing, proven implementation.

And better yet, consider whether you actually need Leader Election at all. Can't you just delegate to your storage provided guarantees?