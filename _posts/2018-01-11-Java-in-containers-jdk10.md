---
layout:     post
title:      People do put Java in containers
date:       2018-01-10 12:00:00
summary:    JDK10 in containers
tags:
- java-TIL
- jdk10
- containers
- draft
---

*TL;DR The JDK team has committed to making Java a good citizen in a world of containers. JDK10 contains several changes to have the JVM and your apps respect container hardware limits.*

This post is a counterpoint or followup to Jörg Schad's recent post [*Nobody puts Java in a container*](https://jaxenter.com/nobody-puts-java-container-139373.html). I would absolutely recommend reading that for its excellent summary of how container technology affects the JVM today.

I can't really agree with his title though - lots and lots of people *do* put JVM workloads in containers (spoiler: I have done so on my last several projects - we even [tune for it](https://github.com/fnproject/fdk-java/blob/master/runtime/Dockerfile-jdk9#L6-L18) in [Fn Project](http://fnproject.io)). As Jörg points out, when JDK10 is released the support will be even better.

There is a [significant amount of work](https://bugs.openjdk.java.net/browse/JDK-8146115) going into JDK10 to support containerized JVMs. In this post we'll see how the next release of the JDK will be container-aware.

I'll be using Docker to run the latest build of JDK10. For the JDK download, head to [http://jdk.java.net/10/](http://jdk.java.net/10/). These patches landed in 10+34. I am using [this Dockerfile](https://gist.github.com/mjg123/cbdee8a9ecba76ec19853d0ac0269d3d). I'll be using a baremetal instance on [Oracle Cloud Infrastructure](https://cloud.oracle.com/en_US/infrastructure/compute) which comes with 72 cores and 256Gb of RAM. Just the kind of place to worry about how best to run lots of things concurrently 😉

## CPU management

Much code will use [`Runtime.getRuntime().availableProcessors()`](http://download.java.net/java/jdk10/docs/api/java/lang/Runtime.html#availableProcessors()) to size thread pools, for example:

  - [core.async](https://github.com/clojure/core.async/blob/d81acd/src/main/clojure/clojure/core/async/impl/concurrent.clj#L28-L30)
  - [ElasticSearch](https://github.com/clojure/core.async/blob/d81acd/src/main/clojure/clojure/core/async/impl/concurrent.clj#L28-L30)
  - [Netty](https://github.com/netty/netty/blob/98beb777f81f092aa0fddf49ed08b426b2c72f01/common/src/main/java/io/netty/util/NettyRuntime.java#L69)
  
  It's a sensible strategy. Even if your code doesn't do so directly, you're pretty likely to be using something which uses the ForkJoinPool under the hood, and [Oh Look!](http://hg.openjdk.java.net/jdk10/jdk10/jdk/file/777356696811/src/java.base/share/classes/java/util/concurrent/ForkJoinPool.java#l2178)
  
Without specifying any constraints, a containerized process will be able to see and use all the hardware on the host. If that's not what you want there are several ways to limit the CPU which a container can use in Docker. Let's see how the value of `availableProcessors` is affected by the different CPU throttling mechanisms:


### Host OS

Running outside of any container, this is the JVM's view of the host.

```shell
$ echo 'Runtime.getRuntime().availableProcessors()' | jdk-10/bin/jshell -q
jshell> Runtime.getRuntime().availableProcessors()
$1 ==> 72
```

### CPU Shares

  Specified with `—cpu-shares` [[Doc](https://docs.docker.com/engine/reference/run/#cpu-share-constraint)].
  
  This rations the CPU according to the proportions you choose, but only when the system is busy. For example you can have three containers with shares of 1024/512/512, but those limits are only applied when necessary. When there is headroom your containerized process can use left-over CPU time.

```shell
$ echo 'Runtime.getRuntime().availableProcessors()' | docker run --cpu-shares 36864 -i jdk10
jshell> Runtime.getRuntime().availableProcessors()
$1 ==> 36
```

Notes:
  1. This is based on the relationship 1 CPU = 1024 'shares'. `36864 = 36×1024`.
  1. That said, 1024 is the default setting, so specifying 1024 here will cause the JVM to report 72 processors rather than 1. This seems like a gotcha waiting to happen.
  1. CPU shares are hard for the JVM or any containerized process trying to do this kind of introspection. The arg you specify is interpreted relative to all the other containers on the system.


### CPU Period/Quota

  Specified with `—cpus` (or `—cpu-period` and `—cpu-quota`, where `CPUs = quota/period`) [[Doc](https://docs.docker.com/engine/reference/run/#cpu-period-constraint)].
  
  This uses the Linux Completely Fair Scheduler to limit the container's CPU usage. This means that the container will have limited CPU time even when the machine is lightly loaded and your workload may be spread across all the CPUs on the host. The example here might give you 50% time on all 72 cores, for example.

```shell
$ echo 'Runtime.getRuntime().availableProcessors()' | docker run --cpus 36 -i jdk10
jshell> Runtime.getRuntime().availableProcessors()
$1 ==> 36
```

### CPU Sets

  Specified with `—cpuset-cpus` [[Doc](https://docs.docker.com/engine/reference/run/#cpuset-constraint)].
  
  Unlike the two previous constraints, this pins the containerized process to specific CPUs. Your process may have to share those CPUs, but will obviously not be allowed to use spare capacity on any others.

```shell
$ echo 'Runtime.getRuntime().availableProcessors()' | docker run --cpuset-cpus 0-35 -i jdk10
jshell> Runtime.getRuntime().availableProcessors()
$1 ==> 36
```

### Combinations

You can mix these options together, too - eg 50% of time on cores 0 through 8. It's well documented by Docker which will link you through to the kernel docs for more info.

### JVM calculation

The formula the JVM uses is: `min(cpuset-cpus, cpu-shares/1024, cpus)` rounded up to the next whole number.

### CPU Limitations Summary




It seems logical to use **CPU Sets** when you have enough cores as it should reduce context-switches and increase CPU cache coherency. Or maybe it would be OK to have more threads, if they spend a lot of their time parked and waiting on an interrupt, in which case perhaps **CPU period/quota** would be appealing. Maybe you are happy with the constraints of **CPU shares** and are happy to be able to use spare CPU cycles. Contrary to the warnings in the JavaDoc I have not seen any occasions where `.availableProcessors()` changes its result over time within a single JVM.

## Memory

The amount of memory that the JVM will try to allocate and make available is either set explicitly or chosen for you by a process known as *Ergonomics*. The docs state that on a 'server-class' machine (>2 processors, >2Gb RAM) the JVM will run in 'server' mode and max heap size will be set by ergonomics to 1/4 physical memory. In fact in 64bit JVMs there is no alternative to 'server' mode. Additionally the heap size chosen by ergonomics is limited to around 32Gb - if you want more than that you have to ask for it with `-Xmx`.

For example, on my test server (256Gb RAM):

```shell
$ jdk-10+23/bin/java -XX:+PrintFlagsFinal -version | grep MaxHeapSize
   size_t MaxHeapSize                              = 32178700288                              {product} {ergonomic}

```

32Gb. It seems that ergonomics isn't designed for such large machines. Anyway, let's see how a memory-limited container environment is treated:

```
$ docker run -it -m512M  --entrypoint bash jdk10
root@7378a1f0a2a9:/# /java/jdk-10/bin/java  -XX:+PrintFlagsFinal -version | grep MaxHeapSize
   size_t MaxHeapSize                              = 134217728                              {product} {ergonomic}
```

This is close enough to 128Mb, as expected.

```shell
$ docker run -it -m512M  jdk10
jshell> Runtime.getRuntime().maxMemory()
$1 ==> 129761280
```

Again, close to 128Mb. No surprises what happens if we try to use too much memory:

```
jshell> new byte[140_000_000]
|  java.lang.OutOfMemoryError thrown: Java heap space
|        at (#1:1)
```


## More Ergonomics

Ergonomics also tunes internal JVM values, such as threadpool sizes used by G1GC. These are reported by the arg `-XX:+PrintFlagsFinal` as `ConcGCThreads` and `ParallelGCThreads` as described on [the OTN](http://www.oracle.com/technetwork/articles/java/g1gc-1984535.html). G1GC threadpool sizes are selected by the same means as `availableProcessors()`.

### Host OS

```shell
$ jdk-10/bin/java -XX:+PrintFlagsFinal -version | grep -i GCThreads
     uint ConcGCThreads                            = 12                                       {product} {ergonomic}
     uint ParallelGCThreads                        = 48                                       {product} {default}
```

### CPU-limited Container


```shell
$ docker run -it --cpus 36  --entrypoint bash jdk10
root@6a94863c54df:/# /java/jdk-10/bin/java -XX:+PrintFlagsFinal -version | grep -i GCThreads
     uint ConcGCThreads                            = 6                                        {product} {ergonomic}
     uint ParallelGCThreads                        = 25                                       {product} {default}
```

### Other values

Several other values are set by the ergonomics process:

  - HotSpot compilation thread count
  - GC region sizes
  - Code cache sizes

Here is [a diff of host vs container values](https://gist.github.com/mjg123/0ed20712de5c686b054ddb49a7f6827d). The container is run with `--cpus 36 -m 4G`.

## Summary

In JDK10 it does seem that applying CPU and memory limits to your containerized JVMs will be straightforward. The JVM will detect hardware capability of the container correctly, tune itself appropriately and make a good representation of the available capacity to your application.

Thanks to [@msgodf](https://twitter.com/msgodf) and Bob Vandette for help with this post.
