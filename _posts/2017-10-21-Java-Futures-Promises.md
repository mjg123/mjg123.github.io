---
layout:     post
title:      Futures and promises in core Java
date:       2017-10-23 17:00:00
summary:    A summary of Futures and promises in core Java
tags:
- draft
- fn-flow
---

This post is a summary of part of a talk I have given a few times about [FnFlow - LINK!!]() internals. I'm writing up the talk as a series of posts of which this is the first.


**Timeline**
* TOC
{:toc}

## Java 5

In 2004 Java 5 was released introducing a load of new features: generics, autoboxing, annotations, and.... `java.util.concurrent`.

### Future

You could use the exciting new `j.u.c.Future` class to perform asynchronous work, although you had to provide your own thread pool for it to run on.

Here's a task which takes some time to run, so we'd like to call it asynchronously:

```java
jshell> String sleepRandomlyThenReturn(String s){
   ...>   try { Thread.sleep ((long) Math.random() * 2000); return s; }
   ...>   catch (Exception e){return s;}
   ...> }
|  created method sleepRandomlyThenReturn(String)
```

And here's us doing that; creating a future by submitting a `Callable` to a threadpool:

```java
jshell> ExecutorService pool = Executors.newFixedThreadPool(1)
pool ==> java.util.concurrent.ThreadPoolExecutor@56ef9176[ ...  = 0, completed tasks = 0]

jshell> Future<String> f = pool.submit(
   ...>   new Callable<String>() {
   ...>     public String call() {
   ...>       return sleepRandomlyThenReturn("Welcome to the Future!");
   ...>     }
   ...>   }
   ...> );        // returns immediately
f ==> java.util.concurrent.FutureTask@29ca901e

jshell> f.get();  // blocks until the callable has finished
$6 ==> "Welcome to the Future!"

jshell> pool.shutdownNow();
$7 ==> []
```

Chaining another task on after the `.get()` was up to you, and if you wanted to do lots of async work it was up to you to manage it all.

## Java 7

### ForkJoinPool

Fast-forward to 2011, [Doug Lea](http://gee.cs.oswego.edu/dl/) adds [`ForkJoinPool`](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/ForkJoinPool.html) to Java, as a best-of-breed work-stealing thread pool. It's possible to submit `Callable` instances to the `ForkJoinPool.commonPool()` and get a `Future` back. Neat.

## Java 8

We've almost caught up with ourselves. Java 8 was released in 2014 and included a big update to the way we can use `Future`. More Doug Lea wizardry: The [`CompletableFuture`](http://download.java.net/java/jdk9/docs/api/java/util/concurrent/CompletableFuture.html).

### CompletableFuture

`CompletableFuture` implements `Future` so everything above still applies. But it also implements a new interface: [`CompletionStage`](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/CompletionStage.html), which is intended to solve the problem I alluded to earlier about composition of async tasks.

### CompletionStage

A `CompletionStage` is, according to its own docs:

> A stage of a possibly asynchronous computation, that performs an action or computes a value when another CompletionStage completes. A stage completes upon termination of its computation, but this may in turn trigger other dependent stages.

In other words, it is a `Future` which can be combined with other `CompletionStage` instances to create a workflow which is defined quite independently of the behaviour of the actual stages themselves.

There are methods which support simple do-this-then-do-that chaining, fan-out, fan-in, error-handling and dynamically adding new stages.

The error handling works like this: if the code running in a stage throws an Exception it doesn't blow up the whole series of CompletionStages. The failed stage is marked as "completed exceptionally" and can be recovered by subsequent stages in a very `try-catch` kind of way.

There's even a way for a stage to add new stages to itself which sounds mind-bending but really allows a whole lot of cool behaviours.

#### A basic example

```java
jshell> cs = CompletableFuture.supplyAsync( () -> sleepRandomlyThenReturn("Hi there") );
cs ==> java.util.concurrent.CompletableFuture@7fac631b[Not completed]

jshell> cs  // it's completed after a few seconds
cs ==> java.util.concurrent.CompletableFuture@7fac631b[Completed normally]

jshell> (cs).thenApply( String::length )
            .thenAccept( (x) -> System.out.println("You were " + x + " chars long") )
You were 8 chars long
```

Notice that there's not many explicit types there. But the API enforces that Stages can only produce values that their dependents can accept - it's pretty smart.

#### CompletionStage as a graph

I have come to think of these chains of CompletionStages as *Execution Graphs* and will refer to them as such liberally. I have a mental image of them something like this:

![A simple completion graph]( {{ "assets/simple-cs.png" | relative_url }} )

#### Properties of a CompletionStage

Each CompletionStage has 4 significant properties:

  * The name (if it has one). The chaining style of the API allows a lot of stages to be anonymous but sometimes you need to actually have a reference to pass somewhere else.
  * Which method was used to create it. This defines the behaviour of the stage.
  * The value, or exception it is completed with.
  * Whether or not it completed successfully, indicated by the colour.


#### Completing Exceptionally

Here's an example with an stage that completes exceptionally:

```java
jshell> cs = CompletableFuture.supplyAsync( () -> { throw new RuntimeException("arrrgh!"); } );
cs ==> java.util.concurrent.CompletableFuture@cd2dae5[Not completed]

jshell> cs    // Completed exceptionally
cs ==> java.util.concurrent.CompletableFuture@cd2dae5[Completed exceptionally]

jshell> cs.thenAccept( System.out::println )
          .exceptionally( (e) -> { System.out.println(e.getMessage()); return null; } )
java.lang.RuntimeException: arrrgh!

```

Which looks something like this, in my head:

![CompletionStage graph with an exception]( {{ "assets/exceptional-cs.png" | relative_url }} )

#### The type-checker bites back

We've offended by the type-checker a bit. The middle stage is of type `CompletionStage<Void>`, which the final stage has to match. If we just try to call `System.out.println` as the lambda in the final stage we get a type error, so we need to explicitly `return null`:

```java
jshell> cs.thenAccept( System.out::println ).exceptionally( (e) -> System.out.println(e.getMessage()) )
|  Error:
|  incompatible types: bad return type in lambda expression
|      void cannot be converted to java.lang.Void
|  cs.thenAccept( System.out::println ).exceptionally( (e) -> System.out.println(e.getMessage()) )
|   
```

*sigh*

#### The method zoo

There are *tons* of methods on the `CompletionStage` API but they break down into groups are are easy to understand in isolation so it's not as bad as it looks.

## Java 9

New CompletionStage methods
