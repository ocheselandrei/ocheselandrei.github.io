---
layout: post
title:  "The book that made me stop worrying about algorithms"
tags: [Algorithms, How to think]
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

I sticked with more takeways from this book than from any other similar book I read. For example, I also went through [The Algorithm Design Manual](https://www.amazon.com/Algorithm-Design-Manual-Steven-Skiena/dp/1849967202). The most relevant thing I still remember from there is the advice to not freeze when presented with a complicated problem. Go through a checklist instead, like plane pilots did when in danger of hitting a quickly approaching mountain. And that is good advice. But I only remembered about that advice while writing this. So the things I read in the How to Think about Algorithms book sticked even more.

## My takeaways from the book

### Loop invariants

Loop invariants are the main way of thinking about iterative algorithms, algorithms that do not use recursion.

Iterative algorithms that use only sequential and conditional statements are easy. The problems come when there are loops, and different iterations of the loop may update a different set of variables or take a different set of actions. This is when the mess begins without using the loop invariant concept.

A loop invariant refers to what stays the same across loop iterations. The thing that stays the same comprises the set of variables maintained by the algorithm and a set of relationships that must hold between those variables. Which is the state of the algorithm. After each iteration, the variables must still mean the same thing and hold the same relationships. So the state must remain consistent. You choose the state conveniently, such that when the loop reaches an exit condition, that state leads directly to the solution. To lead directly to the solution, the state must usually encode a partial solution of the problem as the algorithm progresses. The book presents 3 main types of loop invariants:

* more of the input (after each iteration, you keep the output for the input read so far - many algorithms fit this, default loop invariant to think about)
* more of the output (after each iteration, you add a new element of the output, by re-scanning the input for example - graph shortest path as example)
* reduce the search space (after each iteration, you decrease the range where the output may be - binary search as example)

But for me, a loop invariant is most valuable as a guide on what to spend time thinking about before coding an algorithm:

* What kind of loop invariant does my problem belong to?
* What variables do I need to express the invariant?
* WHAT DO MY VARIABLES REALLY MEAN AND HOW DO THEY KEEP MEANING THE SAME THING BETWEEN ITERATIONS? Think hard about this up-front.

### Think one step at a time

### Trust the process

### Dynamic programming is just an optimization technique

## Disclaimer

I wrote this without re-opening and re-reading the book. It may not be a 100% accurate description of what the book says. But it is how I think about algorithms now after reading the book.
