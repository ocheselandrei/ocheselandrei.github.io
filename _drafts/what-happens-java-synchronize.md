---
layout: post
title:  "What happens when you write synchronized in Java"
categories: Multithreading
---

Started with the simplest way of writing a piece of syncronized code, using a syncronized method:

```java
public class SynchronizedMethodMain {

    public static void main(String[] args) {
        SynchronizedMethodMain synchronizeMain = new SynchronizedMethodMain();
        synchronizeMain.run();
    }

    private synchronized void run() {
    }
}
```

run() bytecode:
```
0 return
```

I knew about some monitorenter/monitorexit stuff, but none of that in run(). Maybe there is something special in main() instead: 

```
 0 new #2 <ocheselandrei/synchronize/SynchronizedMethodMain>
 3 dup
 4 invokespecial #3 <ocheselandrei/synchronize/SynchronizedMethodMain.<init>>
 7 astore_1
 8 aload_1
 9 invokespecial #4 <ocheselandrei/synchronize/SynchronizedMethodMain.run>
12 return
```

Nope, nothing special in main() also.

Maybe something changes if I use a syncronized block instead:

```java
public class SynchronizedBlockMain {
    public SynchronizedBlockMain() {
    }

    public static void main(String[] args) {
        SynchronizedBlockMain synchronizeMain = new SynchronizedBlockMain();
        synchronizeMain.run();
    }

    private void run() {
        synchronized(this) {
        }
    }
}
```

run() bytecode:

```
 0 aload_0
 1 dup
 2 astore_1
 3 monitorenter
 4 aload_1
 5 monitorexit
 6 goto 14 (+8)
 9 astore_2
10 aload_1
11 monitorexit
12 aload_2
13 athrow
14 return
```

So now we're getting one monitorenter and 2 monitorexits. 2 monitorexits is expected, one for happy case, one for exceptions. But why is there a difference between syncronized methods and syncronized blocks? Did the compiler just decide that there are no multiple threads to access the syncronized method and optimized out the syncronization code?

Going to the theory, and nope, none of that optimization stuff. Java Language Specification for [synchronized](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-3.html#jvms-3.14) says that syncronized methods are to be implemented differently, by setting an ACC_SYNCHRONIZED flag in the run-time constant pool instead of a monitorenter/monitorexit pair.

Confirmed by checking the run() methods access flags:

```
Synchronized method run() access flags: 0x0022 - private syncronized
Synchronized block run() access flags: 0x0002 - private
```
