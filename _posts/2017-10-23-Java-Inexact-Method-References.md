---
layout:     post
title:      JavaTIL Inexact Method References
date:       2017-10-23 12:31:19
summary:    What even are "Inexact" Method References?
---

This is a quick post of **JavaTIL**. Todays topic inspired by a [Twitter post](https://twitter.com/joshbloch/status/921881630809014272) from Josh Bloch asking why the following Java code doesn't compile:

```java
void puzzler() {
    ExecutorService exec = Executors.newCachedThreadPool();
    exec.submit(System.out::println);
}
```

First of all, I'll take Josh's word for it. But it looks to me like it *could* compile - `System.out.println` invoked with no args will do some stdio and return null, so `exec.submit()` can return a `Future<Void>`. Hmmm.

Lets see the error then. I don't usually role-play as a compiler when I have a good one to hand in `jshell`:

```java
jshell> ExecutorService exec = Executors.newCachedThreadPool();
exec ==> java.util.concurrent.ThreadPoolExecutor@56ef9176[ ...  = 0, completed tasks = 0]

jshell> exec.submit(System.out::println);
|  Error:
|  reference to submit is ambiguous
|    both method <T>submit(java.util.concurrent.Callable<T>) in java.util.concurrent.ExecutorService and method submit(java.lang.Runnable) in java.util.concurrent.ExecutorService match
|  exec.submit(System.out::println);
|  ^---------^
|  Error:
|  incompatible types: cannot infer type-variable(s) T
|      (argument mismatch; bad return type in method reference
|        void cannot be converted to T)
|  exec.submit(System.out::println);
|  ^------------------------------^
```

Two errors. The first is confusing - hence it being a puzzle ;-) Here it is again:

```
both method <T>submit(java.util.concurrent.Callable<T>) in java.util.concurrent.ExecutorService
 and method submit(java.lang.Runnable) in java.util.concurrent.ExecutorService match
```

The compiler is complaining that it can't tell whether method reference `System.out::println` should have target type of `Runnable` or `Callable`.

But in other circumstances it can clearly tell that only `Runnable` matches:

```java
jshell> Runnable r = System.out::println    // YEP!
r ==> $Lambda$17/267760927@25bbe1b6

jshell> Callable c = System.out::println    // NOPE!
|  Error:
|  incompatible types: bad return type in method reference
|      void cannot be converted to java.lang.Object
|  Callable c = System.out::println;
|               ^-----------------^

jshell> Callable<Void> c = System.out::println // ALSO NOPE!!
|  Error:
|  incompatible types: bad return type in method reference
|      void cannot be converted to java.lang.Void
|  Callable<Void> c = System.out::println;
|                     ^-----------------^
```

So that's weird, isn't it?

Fiddling about with this it seems to be triggered by `System.out::println` being variadic. Here's a minimal repro:

```java
jshell> class Banana { void foo(){}; void foo(int s){} }
|  created class Banana

jshell> exec.submit(new Banana()::foo)
(same error as before)
```

## The Answer

As figured out by [@lhochstein](https://twitter.com/lhochstein/status/921913966925721600):

There is a concept of **exactness** which can be applied to a method reference. Quoting [JLS 15.13.1](https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.13.1):

> For some method reference expressions, there is **only one** possible compile-time declaration with **only one** possible invocation type, regardless of the targeted function type. Such method reference expressions are said to be **exact**. A method reference expression that is not exact is said to be **inexact**.

Given this, it's clear that `System.out::println` is **inexact** as there are 10 overloads.

[Elsewhere in the JLS](https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.12.2.2) we find that inexact method references are excluded from being considered in method overload resolution:

> An argument expression is considered pertinent to applicability for a potentially applicable method m unless it has one of the following forms:
>    [...]
>    An inexact method reference expression (§15.13.1).


## Exactly...

Method references don't work *quite* how I thought they did. JDK8 improved a lot of things, but in Josh's words "Compromises were required". [Here](http://bugs.java.com/bugdatabase/view_bug.do?bug_id=JDK-8144169) and [here](https://bugs.openjdk.java.net/browse/JDK-8176576) people have raised this behaviour as being a bug.
