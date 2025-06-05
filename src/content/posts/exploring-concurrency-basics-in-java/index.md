---
id: 'c1a4f671-4c9a-4568-8fa4-58ae365c6069'
title: 'Exploring Concurrency: Basics in Java'
description: 'An introduction to concurrency in Java. Join me as I explore the basics, potential pitfalls and advantages.'
date: '2018-04-04 21:52:00'
tags:
  - Concurrency
  - Java
---

_This is the first part of my 'Exploring Concurrency' series._

As a web developer, I spend most of my time working in a single-threaded environment. I would say this is a fairly safe
assumption for most other web developers as well. Ultimately, it's much easier to write code when you don't have to
worry about your code producing weird results and locking up indefinitely.

Recently I've decided to peer into Pandora's Box and learn more about concurrent programming and how I can incorporate
it into my toolkit. As part of that learning I am going to write up my findings in a series of posts to consolidate
some of that knowledge in a useful way.

For most of this series, I will mostly be working with the JVM languages (perhaps with some other novel languages
thrown in here and there). Let's kick things off by exploring the basics using the most popular option - Java.

## The Thread primitive

As you might expect from Java, the out-of-the-box concurrency primitives are simply OOP abstractions. To work with a
thread, you simply create a new `Thread` and pass it a `Runnable` method. With Java 8 Lambda syntax this looks like:

```java
new Thread(() -> System.out.println("Thread finished")).start();
```

Of course, if you need object-like behaviours, then you just need to implement `Runnable` or extend `Thread` and use
that implementation instead:

```java
new Thread(new Runnable {
  private int counter = 0;

  @Override
  public void run() {
    counter++;
    System.out.println("Thread finished");
  }
})
  .start();
```

Threads can be interrupted to signal that they should do some other work, or simply stop. For example:

```java
Thread childThread = new Thread(() -> {
  try {
    Thread.sleep(4000L);
    System.out.println("Child thread finished");
  } catch (InterruptedException e) {
    System.out.println("Child thread interrupted");
    // Propagate interrupt back up to the parent thread
    Thread.currentThread().interrupt();
  }
});

childThread.start();

// Sleep main thread then interrupt child prematurely
Thread.sleep(2000L);
childThread.interrupt();
```

Outputs after a 2 second delay:

```
Child thread interrupted
```

Methods such as `sleep` have a checked `InterruptedException`. The reasoning for this is that the language designers
wanted to force users to handle such interruptions all the way back up the call stack, which seems reasonable.
Unfortunately this leads to a lot of noise in our code (similarly for most checked exceptions). I think it would be
good if we could attach some kind of exception handler to `Thread` instead, like the following pseudo-code:

```java
new Thread(
  () -> Thread.sleep(4000L),
  e -> Thread.currentThread().interrupt()
);
```

Unfortunately, I doubt this kind of API change will happen any time soon! For brevity, I will not be including any
further unnecessary checked exception handling for the rest of this post.

Continuing on, threads can be made to wait until other threads are completed before continuing execution by using
`join`.

```java
Thread childThread = new Thread(() -> {
  Thread.sleep(2000L);
  System.out.println("Child thread finished");
});

childThread.start();
childThread.join();

System.out.println("Main thread finished");
```

Outputs:

```
Child thread finished
Main thread finished
```

You might expect the opposite ordering as the `childThread` has to sleep for 2 seconds first. However, due to the
`childThread.join` call, the main thread waits for `childThread` to complete before continuing with its own execution.
Fairly straightforward so far.

## Ready, set... go!

Whilst the basic usage of threads is pretty simple, the reality of using multiple threads can quickly become quite
messy and unintuitive. We'll discuss the major problems that most concurrent code will have to face.

### Thread interleaving

The first major issue is that threads do not have any guarantees on the order that they will run. This means we can get
weird and wonderful results from innocuous looking code:

```java
int x = 0;

new Thread(() -> x++).start(); // Thread A
new Thread(() -> x++).start(); // Thread B

assertThat(x).isEqualTo(2); // Could be true OR false
```

Whoa what? How can `x` be 1? It's like one operation was completely disregarded!

We would logically expect that:

1. Thread A increments `x` before thread B.
2. Thread B picks up the most recent increment of `x`.
3. Thread B then increments `x` again using this up-to-date value.

As it turns out, threads can be scheduled (internally) to execute at any time. This means that you can end up in a
scenario where they execute simultaneously (or overlap) and both see the value of `x` as 0. When this happens, they both
set `x` to 1 instead of the expected 2.

Life gets even harder if we increase the number of threads. We increase the number of permutations and make the results
even more unpredictable. Using three threads, `x` could equal 1, 2 or 3 depending on the order they run. Clearly not
great news if we want to have bug-free code!

This issue is due to **thread interleaving** and is at odds with our normal, sequential thinking. A popular
expression for this issue is the term - **race condition**.

To remedy this we need our code to be **atomic**. We need to ensure that when we perform compound operations they are
actually performed in a single operation that is synchronized between threads.

#### More on compound operations

Typically compound operations consist of a read operation followed by some action i.e. 'check-then-act' or
'read-modify-write' operations. The differences between these types of compound operation are subtle, but worth thinking
about. The main difference is that 'read-modify-write' expects that an update operation is performed using the most
recent read operation's value.

The incrementation example earlier is a 'read-modify-write' operation. We get the most recent value, increment it then
try to update the corresponding variable. The intention was to update `x` sequentially in two increments, however one
of the updates was invalidated by the second thread's update.

On the other hand, 'check-then-act' usually revolves around the potential for a condition to pass when it shouldn't
due to using a stale value. For example:

```java
boolean doAction = true;

Runnable action = () -> {
  if (doAction) {
    System.out.println("Action was ran.");
    doAction = false;
  }
}

new Thread(action).start();
new Thread(action).start();
```

Can print the following:

```
Action was ran.
Action was ran.
```

Clearly our intention was only for 'Action was ran.' to be printed once, but here we can see it twice. You can imagine
how this could be very unwarranted in a production application!

### Memory consistency

Our problems don't stop there. We also have another problem which is even weirder.

Can we guarantee that the threads see the correct state of the resource at the correct moment in time during execution?

Unfortunately not! Changes to shared state are not guaranteed to ever be made visible to other threads that also have
access to the shared state. This means that you can have peculiar situations like the following:

```java
boolean looping = true;

new Thread(() -> {
  Thread.sleep(100L);
  looping = false;
})
  .start();

new Thread(() -> {
  while (looping) {
    // Do some work
  }

  System.out.println("Thread 2 finished");
})
  .start();
```

Again this looks innocent enough, but you might be surprised to find out that thread 2 may never finish looping. The
update to `looping` by thread 1 is never made available to thread 2, so we get stuck in a continuous loop.

This time the problem is one of **memory consistency**. There are no guarantees that a thread's local memory will be
flushed back to the main memory (shared by other threads) in a timely fashion. Another way of expressing this would be
to say there is insufficient **visibility** between threads.

## Teaching threads to work together

As we can see there are some tricky problems that we face to take advantage of having those extra threads.

Thankfully there are a few immediate features we can use to get threads to coordinate their efforts correctly.

### The `sychronized` keyword

The easiest way of controlling the activity multiple threads is to use the `synchronized` keyword. It can be attached to
class methods allowing only one thread to access the method at a time. For example:

```java
class SyncedCounter {
  int counter = 0;

  synchronized inc() { counter++; }
}

SyncedCounter counter = new SyncedCounter();

new Thread(() -> counter.inc()).start();
new Thread(() -> counter.inc()).start();

assertThat(counter.counter).isEqualTo(2);
```

It can also be applied in statements like the following for more 'fine-grained' control over specific blocks of code:

```java
int counter = 0;
Object lock = new Object();

new Thread(() -> {
  synchronized (lock) {
    counter++;
  }
})
  .start();

new Thread(() -> {
  synchronized (lock) {
    counter++;
  }
})
  .start();

assertThat(counter).isEqualTo(2);
```

Note that in this instance, we also have to manually provide a `lock` object. If the thread has access to that lock
object, then only it can execute the code block - it is said to have acquired the lock. Typically we could just pass in
`this` (instead of some other object) as the lock for convenience. Using the `synchronized` keyword on class methods
directly does the same thing, but automatically uses `this` as the lock.

We can see that synchronization is able to solve our **thread interleaving** and **memory consistency** problems.
Unfortunately, we cannot apply it wholesale as it works against the performance benefits of using multiple threads in
the first place. Synchronization is expensive and can cause **thread starvation** - a situation where threads sit
idle when they could be better utilised elsewhere. Consequently, the ultimate aim should be to limit the scope of every
`synchronized` block to be as small as possible.

One thing to note - whenever you use `synchronized` around some shared state, you need to ensure that ALL access (read
or write) to the shared state is synchronized to ensure thread safety.

### The `volatile` keyword

Another mechanism we can employ is the use of the `volatile` keyword on class fields. This allows us to get around the
problem of **visibility** that we saw earlier. With this keyword, updates to the class field will be made immediately
visible to other threads (rather than potentially never).

Augmenting our previous example to avoid visibility problems:

```java
class LoopChecker {
  // Volatile can only be added to instance members
  volatile boolean looping = true;
}

LoopChecker loopChecker = new LoopChecker();

new Thread(() -> {
  Thread.sleep(100L);
  loopChecker.looping = false;
})
  .start();

new Thread(() -> {
  while (loopChecker.looping) {
    // Do some work
  }

  System.out.println("Thread 2 finished");
})
  .start();
```

This time it actually prints the following:

```
Thread 2 finished
```

`volatile` can be considered a 'lightweight' form of synchronization. However, it is not a complete replacement for
`synchronized` as we can still run into problems where we require **atomicity** e.g. increments/decrements on numbers.

### Atomic classes

The `java.util.concurrent` package provides [atomic versions](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/atomic/package-summary.html) of primitives such as `AtomicBoolean`,
`AtomicInteger`, `AtomicReference`, etc. These classes are designed to provide both **atomicity** and **visibility**
when working with their equivalent primitive types. For example:

```java
AtomicInteger counter = new AtomicInteger(0);

new Thread(() -> {
  System.out.println(counter.getAndIncrement());
})
  .start();

new Thread(() -> {
  System.out.println(counter.getAndIncrement());
})
  .start();
```

Prints the following:

```
1
2
```

Normally without any synchronisation, it could be possible that both printed lines are equal to 1. Fortunately
`getAndIncrement` is an atomic operation ensuring we get the desired output. This atomicity is actually achieved via a
low-level native '[compare-and-swap](https://en.wikipedia.org/wiki/Compare-and-swap)' mechanism that is _not_
synchronization-based. The update operation will only be performed if the current value is _exactly the same_ as an
expected value. In atomic classes, this is exposed as the `compareAndSet` method, which is used under-the-hood
for most (if not all) atomic operations in the atomic classes.

For example, `getAndIncrement` can be simplified to the following pseudo-code:

```java
int current;
do {
  current = get();
} while (!compareAndSet(current, current + 1));
```

This loop essentially calls the atomic `compareAndSet` operation continuously with the 'most recent' value (obtained
from `get`) until it succeeds. I think it's quite interesting that some programming problems are 'best' solved simply
by using brute-force.

In summary, atomic classes should be considered as wrappers around volatile variables, and they are certainly a great
option if we require atomicity. If not, then just using `volatile` may be sufficient, however, it seems that
atomic classes are usually more versatile and useful.

### Collections

As working with collections is commonplace in nearly any application, there are two packages we can immediately reach
for when attempting to make our collections thread-safe.

#### Synchronized collections

These are actually just wrapper classes that add `synchronized` to every public method of the internal collection.
They are created by calling their respective static methods:

```java
Collections.synchronizedList(new ArrayList<Integer>());
Collections.synchronizedMap(new HashMap<String, Integer>());
Collections.synchronizedSet(new HashSet<Integer>());
```

As these collections just add `synchronized` to every public method, they do not exhibit good concurrent performance as
only one thread will be able to access the collection at a time. As we've discussed earlier, this can cause
**thread starvation** and is not scalable in a multi-threaded environment. These synchronized collections _may_
be useful in certain scenarios, but there are other types of collections that should be considered first.

#### Concurrent collections

These are available via the `java.util.concurrent` package and are designed to solve the performance deficiencies in
the synchronized collections. Examples of equivalent concurrent collections include `ConcurrentHashMap`,
`CopyOnWriteArrayList` and `CopyOnWriteArraySet`.

So how do these concurrent collections differ?

Let's first look at `ConcurrentHashMap`. This implementation uses a more fine-grained locking mechanism called
_lock-striping_ to allow greater shared access between threads (instead of a single lock on all public methods). This
allows many threads to read from the map, and a _limited number_ of threads to write to it concurrently. Consequently,
the `get`, `put`, `containsKey` and `remove` methods are well-optimized for concurrency.

Unfortunately, these optimizations are not without trade-offs. Operations such as `size` and `isEmpty` are purposefully
not guaranteed to be accurate. The collection size is considered to be a 'moving target' - definitely something to be
aware of!

As `ConcurrentHashMap` is not designed for exclusive locking (like the synchronized version), it exposes helpful atomic
operations such as `putIfAbsent`, `remove` and `replace`.

Looking at `CopyOnWriteArrayList/Set`, these collections provide thread safety by copying the entire collection into a
new collection every time there is an update. Iterators on this collection are constructed only with a snapshot of the
collection at that point in time. However, this also means that later updates will not be propagated down to any of the
iterators that already exist.

These collections are effectively **immutable** avoiding the need for heavyweight synchronization methods. Again, this
is not without trade-off. Copying large collections is not fast so we should avoid using these collections when updates
are more common than read-only iterations.

## Conclusion

Concurrency is an attractive method for us to squeeze more performance out of our code. Unfortunately it is not for free
as there are caveats that are not obvious and we have to be careful with.

- Threads can **interleave** meaning that that operations are executed in unpredictable orders. When using some shared
  state, this can result in unexpected results.
- Threads are not guaranteed to make their local changes to shared state available to other threads. This causes
  **memory consistency** issues.

These issues can be very problematic as their effects can be very subtle and dangerous. To battle these issues, we can
employ several language features:

- Use the `synchronized` keyword. This should be made as fine-grained as possible to avoid introducing performance
  problems (which defeats the point of multi-threading our code).
- Use the `volatile` keyword. This allows us to flush updates to variables back to main memory immediately and make
  them visible to other threads.
- Use the Atomic classes. These are like `volatile` variables, except that they also provide **atomicity** via methods
  such as `compareAndSet`.
- Use concurrent collections. These collections maximize concurrent performance whilst also providing thread safety with
  relatively little overhead (and some caveats). They should be preferred over synchronized collections.

Hopefully this has been a relatively gentle introduction to concurrency in Java. I've tried my best to summarise some of
the raft of information on the topic, but there is plenty more to discuss! For further reading I would recommend
checking out [Java Concurrency in Practice](http://jcip.net/), which is a very comprehensive resource that is still
very relevant to this topic.

In the next post of this series, I will be discussing some of the higher-level abstractions that are available for
dealing with concurrency in the Java standard library. I look forward to seeing you there!
