---
layout: post
title:  "The book that made me stop worrying about algorithms"
tags: [Influential Books, Algorithms]
---
While reading through the Medium newsletters I receive, I noticed a theme about people describing books that influcenced them, and realized I have such a book to share as well.

## Background

At some point after I started my proffesional career, I went searching for resources about better understanding algorithms. Why did I do that?

I was not satisfied with the explanations I found for more complex algorithmic problems, starting from the textbook descriptions during high-school and college. Each explanation seemed coming from a totally different thinking pattern, and it just jumped into the solution: this is how to represent the input, the output, the steps between them. But it was not clear how could a regular person invent such a solution, generalize such a solution to multiple problems.

I was not satisfied with the process of writing my own algorithms. While I ended-up with correct solutions eventually, the process to get there was messier than I would have wanted (lots of run, error, fix, repeat). It wasn't a process filled with confidence, but with worry: does it really work for all cases, will I still find a solution for the next problem?

So, I felt like there had to be a more "deterministic" way of understanding and designing an algorithm. After some searching, I found just that. A book called "[How to Think about Algorithms](https://www.amazon.com/Think-About-Algorithms-Jeff-Edmonds/dp/0521614104)", by [Jeff Edmonds](https://www.cambridge.org/ro/academic/subjects/computer-science/algorithmics-complexity-computer-algebra-and-computational-g/how-think-about-algorithms?format=PB&isbn=9780521614108#bookPeople).

## The [How to Think about Algorithms](https://www.amazon.com/Think-About-Algorithms-Jeff-Edmonds/dp/0521614104) book

This is a book with children pictures in it and is intended as course material for undergrad college students. So some "serious" algorithm researchers would probably not consider looking at or recommending it.

But for me, it was the most influential book on algorithms I ever read. It shaped the way I currently think about algorithms. It made me stop feeling anxious and impressed when someone mentions a fancy algorithm. I think those undergrad students that need to study this book are lucky.

I sticked with more takeways from this book than from any other similar book I read. For example, I also went through [The Algorithm Design Manual](https://www.amazon.com/Algorithm-Design-Manual-Steven-Skiena/dp/1849967202). The most relevant thing I still remember from there is the advice to not freeze when presented with a complicated problem. Go through a checklist instead, like plane pilots did when in danger of hitting a quickly approaching mountain. And that is good advice. But I only remembered about that advice when writing this. So the things I read in the How to Think about Algorithms book sticked even more.

## My takeaways from the book

### Loop invariants

Loop invariants are the main way of thinking about iterative algorithms, algorithms that do not use recursion.

Iterative algorithms that use only sequential and conditional statements are easy. The problems come when there are loops, and different iterations of the loop take a different set of actions on a different set of variables. This is when the mess begins without using the loop invariant concept.

A loop invariant describes the things that don't change about your algorithm state. The algorithm state is the set of variables maintained across the iterations. The loop invariant encodes the meaning of your variables and the relationships that hold between the variables. While your variable values will change, your invariant will not change (duh) from one iteration to the next. When your loop reaches an exit condition, the state maintained according to your invariant gets you directly to the desired output. You get directly to the desired output by making your loop invariant belong to one of these categories described by the book:

* More of the input
  * Your state encodes a complete output for the partial input read so far. At each iteration, you read one more input element and update the output. Exit when input no longer has elements.
  * Most of the algorithms you need to write on a daily basis fit here, so it is the default loop invariant to start with.
* More of the output
  * Your state encodes a partial output. At each iteration, you add one more element to the partial output. You may have to re-scan the input to do that. Exit when you (somehow) know that the output can no longer receive elements.
  * Graph shortest paths as example. Trickier loop invariant type.
* Reduce the search space
  * Your state encodes the search space where your target may be in. At each iteration, you decrease that search space. Exit when the search space is reduced to the target (or nothing).
  * Binary search as example

But for me, a loop invariant is most valuable as a guide on what to spend time thinking about before coding an algorithm:

* What kind of loop invariant does my problem belong to? Default to more of the input.
* What variables do I need to express the invariant?
* WHAT DO MY VARIABLES REALLY MEAN AND HOW DO THEY KEEP MEANING THE SAME THING BETWEEN ITERATIONS? Think hard about this one.

### Think one step at a time

The book describes 2 main classes of algorithms: iterative and recursive.

For iterative algorithms, you start from the middle of your computation. You somehow landed at the start of one intermediate step (iteration). You have your loop invariant. You know what your variables mean and the relations they hold at the beginning of the step. Your job is to only think about how to move to the next step. That's it. That's all the mental state you need to maintain. No worrying about initial values, initial edge cases, whether you can actually land at the beginning of that step, etc. These can be postponed to when the algorithm becomes clearer. And some of those edge cases may even be naturally handled by the algorithm anyway.

For recursive algorithms, thinking one step at a time becomes even more important. You are a function handling one instance of your problem. The functions handling smaller instances of your problem are your friends. You can trust your friends that they will do you what you expect. Of course, you can't expect your friends to do things you can't do (you are also a friend after all). So your job is to only think about the direct calls you make to your friends to get an answer to your problem. That's it. No mental tracing through the entire recursion tree. If you feel like you need to trace the entire tree, then you are not clear on what your friends (and you) must do. Spend time clarifying that instead. Now that can be easier said than done, but it's worth giving it a try.

### Trust the process

You need a leap of faith that inventing a loop invariant and thinking one step at a time will work instead of tracing all the possible states from the beginning of your iterative computation.

You need a leap of faith that focusing on just the immediate calls your recursion function needs to make will work instead of tracing the entire recursion tree.

You need to resist old habits of trying to keep all state mutations in your mind. But it is worth it, as mental energy will be better spent on the things that matter thinking upfront and hard about.

### Dynamic programming is just an optimization technique

I particularly liked the clarifications that the book brings related to dynamic programming.

The book clarifies that dynamic programming is an optimization for recursive back-tracking. Now this still sounds complicated. In fact, dynamic programming is just an optimization for implementing a recursive algorithm in general, if possible.

You do not start by thinking: How do I find a smart dynamic programming solution to this problem? No. You start by trying to frame your problem as a recursive problem. What are the sub-problems, the smaller instances of your problem? How can you combine the solutions to the sub-problems into a solution for the problem? What is the recurrence relation between the solutions of the problem and the solutions of the sub-problems? This is less intimidating and I would say more interesting to start with.

Say you found a recurrence relation (formula). You could just write your algorithm according to it. But you may notice you need to re-compute solutions to smaller problems. You may notice that the solutions to the sub-problems do not depend on the surrounding bigger problem. Then you could just cache the solutions to smaller problems. Or, you could come-up with a smart way to order the solutions to the sub-problems such that when you get to a bigger problem you find the solutions to the smaller problems already computed. And you may use less memory than the cache in the process. And look, dynamic programming. But it all starts from just solving a problem in regular recursive terms.

## Closing notes

I wrote this without re-opening and re-reading the book. It may not be a 100% accurate description of what the book says. But it is how I think about algorithms now after reading the book.

In the next article, I will follow-up with a practical example of applying the above principles. We will look at the use-case of a problem regularly used in programming interviews, see how existing solutions break the principles, and come-up with both iterative and dynamic programming solutions according to these principles.
