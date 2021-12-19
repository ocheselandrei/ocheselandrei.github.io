---
layout: post
title:  "How To Think About Algorithms - Practical Example"
tags: [Influential Books, Algorithms]
---
This is a practical application of the algorithm design principles from the "How To Think About Algorithms" book. I introduced a bit of theory for the principles in the previous blog post: [The book that made me stop worrying about algorithms]{% post_url 2021-11-21-the-book-that-made-me-stop-worrying-about-algorithms.md %}.

I will use as a starting point a regularly encountered interview problem and its solution from a well-known programming interviews preparation book: [Cracking the Coding Interview](https://www.amazon.com/Cracking-Coding-Interview-Programming-Questions/dp/0984782850). The problem and solution are copied here from the paper copy I own.

### The problem

> You are given an array of integers (both positive and negative). Find the contiguous sequence with the largest sum. Return the sum.
>
> EXAMPLE
>
> Input: 2, -8, 3, -2, 4, -10
>
> Output: 5 (i.e, {3, -2, 4})

As usual with coding problems, you may pause a bit and try to come with a solution on your own.

### The solution in Cracking the Coding Interview book

```
int getMaxSum(int[] a) {
    int maxsum = 0;
    int sum = 0;
    for (int i = 0; i < a.length; i++) {
        sum += a[i];
        if (maxsum < sum) {
            maxsum = sum;
        } else if (sum < 0) {
            sum = 0;
        }
    }
    return maxsum;
}
```

"Cracking the Coding Interview" is a great book with great explanations for the problems. It does offer a textual explanation for this algorithm. But just by looking at the code, several things are not clear.

The `maxsum` variable is clear. But what does the `sum` variable stand for? It represents some kind of running sum. But what does this running sum mean? Where and when does it start, does it end?

If `sum` drops below 0 - `if (sum < 0)`, why is it reset to 0 - `sum = 0`? This breaks consistent definitions of the `sum` variable. What if the array has only negative numbers? This code will return 0, while I would expect it to return the minimum negative number. The book does make a point that for negative numbers only, the empty sequence could be considered the solution, because it has sum 0, which is interesting...

But probably your interviewer will say: "No, empty sequence is not the solution for negative numbers only". How do you adjust the above code to the new requirement?

This is an example of a coding solution that is hard to understand logically. It makes you feel "ungrounded" (a good term used by the How To Think About Algorithms book). It is not clear what the variables really mean and why they are changed the way they are. 

This example shows that even for people well versed in the industry, implementing an algorithm can still be a messy process. My own solutions to this problem looked a lot like the above code. And I implemented this problem at least twice. Every time I forgot about what I did in the previous solution. And it took a lot of time to re-discover how to update that running sum in a way that somehow makes sense and then code it. But that was before I really followed the steps from the "How To Think About Algorithms" book.

### The iterative solution using principles in "How To Think About Algorithms"

#### Loop invariant

What type of loop invariant? We'll start with te default one, the More of the Input. At each iteration, you read one more input element and update the output. Exit when input no longer has elements. You must be able to provide a solution after each partial input (each input prefix).

What are the variables making up the loop invariant? To provide a solution after each step, you must obviously maintain the maximum sum so far. But is this enough?

You are now reading a new element from the input. How do you connect it with the maximum sum so far? The maximum sum sequence could be in the middle of the previous input. So you need an auxiliary information, something that lets you connect the new input element with the previously read input by re-using state maintained so far instead of re-traversing the input.

Chances are, you think of maintaining the maximum sum ending in the last element of the input read so far. This should let you easily extend it to the next element in the input. And then the maximum sum would just be the maximum of all these maximum sums ending at an element in the input. You are now solving more than the original problem, but that's how things are.

So what is the loop invariant? The algorithm will maintain the overall maximum sum and the maximum sum ending in the last element for each input prefix. The maximum sum will at all times be the greatest of all the maximum sums ending in the last element of input prefixes.

#### One step at a time

So how do you maintain the loop invariant at each iteration?

Notation:

* msLast - maximum sum ending in last element of input read so far
* A[i] - new element read from input

Let's see the cases:


| msLast at i-1 | A[i] | msLast at i            | Notes                                                                                                                              |
| --------------- | ------ | ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------ |
| > 0           | > 0  | msLast = msLast + A[i] | Obvious                                                                                                                            |
| > 0           | < 0  | msLast = msLast + A[i] | The maximum sum ending at i benefits from adding the positive maximum sum ending at i-1                                            |
| < 0           | > 0  | msLast = A[i]          | The maximum sum ending at i is reset to also start at i because it would decrease by adding the negative maximum sum ending at i-1 |
| < 0           | < 0  | msLast = A[i]          | Same reasoning as above, even if A[i] is now negative                                                                              |

So in fact there are just 2 cases:

* msLast <= 0 => msLast = A[i]
* msLast > 0 => msLast = msLast + A[i]

Of course, after each msLast update, you update the overall maximum sum if it is lower.

Once you designed the loop invariant, and designed the single step needed to maintain the loop invariant between iterations, writing the code for the algorithm becomes a mechanical operation. It is almost not worth adding the code, but will put it here for reference anyway.

#### The code

```
int maxSum(int[] A) {
    //setting up the loop invariant variables
    //the overall max sum for input read so far - the output we need
    int ms = 0;
    //the max sum for the sequence ending in the last element of input read so far
    int msLast = 0;

    for (int i = 0; i < A.length; i++) {
        if (msLast <= 0) {
            msLast = A[i];
        } else {
            msLast += A[i];
        }
        ms = Math.max(ms, msLast);
    }
    return ms;
}
```

### The dynamic programming solution using principles in "How to Think About Algorithms" book

So now your interviewers says: "But I wanted a cool dynamic programming solution!".

What would be the steps? Do you start thinking of arrays and what could you put in them? No.

#### Frame the problem recursively

What are the sub-problems? Naturally, you can consider the prefixes of the input.

How to solve a bigger problem from sub-problems?

For reasons presented in the iterative case, just returning the overall maximum sequence sum from a sub-problem is not enough. You need to also return the maximum sum of the sequence ending in the last element of the sub-problem input. You can return in fact just this maximum sum ending at last input element, and then you could just run a max over the solutions for each sub-problem.

What would be the recurrence relation? Using notation and findings from the iterative case:

```
msLast(i) = if (i == 0 || msLast(i-1) <= 0) A[i] else A[i] + msLast(i-1)
```

So you could just compute each msLast(i) according to the recurrence formula, memorize it in its own extra array element, then do a max over that array, and done.

#### Optimize

But you can do better than this. When you compute msLast(i), you don't want to re-compute msLast(i-1) to msLast(0), as you also computed them when computing msLast(i-1)...

So you notice you can just compute the msLast's in increasing order of i's to prevent re-computation and keep them in that extra array you thought of initially, but now it also serves as a cache. And then you notice that you don't actually need an extra array, because you just need the previous value of msLast in order to compute the next value of msLast, and you can just update the overall maximum sum after each update...

And what do you know? You got back to exactly the iterative solution... So you already coded the dynamic programming solution. That is cool to point to your interviewer.

This duality between More of the Input Loop Invariants and Dynamic Programming solutions is a cool thing and the book provides further examples for those interested. If you can't come up with an elegant iterative solution to your problem, you can think how you would solve it recursively, and vice-versa.

### Conclusion

I hope this gave you a solid reason to start designing your algorithms according to the principles in "How To Think About Algorithms" book. You avoid the mess and gain clarity into how your solutions work. Your code becomes obviously correct.
