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

This post is a kind of counterpoint or add-on to Jörg Schad's recent post [*Nobody puts Java in a container*](https://jaxenter.com/nobody-puts-java-container-139373.html). I would absolutely recommend reading that for its excellent summary of how container technology affects the JVM today.

I can't really agree with his title though - lots and lots of people *do* put JVM workloads in containers (spoiler: I have done so on my last several projects - we even [tune for it](https://github.com/fnproject/fdk-java/blob/master/runtime/Dockerfile-jdk9#L6-L18) in [Fn Project](http://fnproject.io)). As Jörg points out, when JDK10 is released the support will be even better.

There is a [significant amount of work](https://bugs.openjdk.java.net/browse/JDK-8146115) going into JDK10 to support containerized JVMs. In this post we'll see how the next release of the JDK will be container-aware.

I'll be using Docker to run the latest build of JDK10. For the JDK download, head to [http://jdk.java.net/10/](http://jdk.java.net/10/). This will be in OpenJDK 10 as well, but I haven't found a recent enough build - these patches landed in 10+34. I am using [this Dockerfile](https://gist.github.com/mjg123/cbdee8a9ecba76ec19853d0ac0269d3d). I'll be using a baremetal instance on [Oracle Cloud Infrastructure](https://cloud.oracle.com/en_US/infrastructure/compute) which comes with 72 cores and 256Gb of RAM. Just the kind of place to worry about how best to run lots of things concurrently 😉

### CPU management

Many apps will use [`Runtime.getRuntime().availableProcessors()`](http://download.java.net/java/jdk10/docs/api/java/lang/Runtime.html#availableProcessors()) to size thread pools, for example:

  - [core.async](https://github.com/clojure/core.async/blob/d81acd/src/main/clojure/clojure/core/async/impl/concurrent.clj#L28-L30)
  - [ElasticSearch](https://github.com/clojure/core.async/blob/d81acd/src/main/clojure/clojure/core/async/impl/concurrent.clj#L28-L30)
  - [Netty](https://github.com/netty/netty/blob/98beb777f81f092aa0fddf49ed08b426b2c72f01/common/src/main/java/io/netty/util/NettyRuntime.java#L69)
  
  ... and so on. It's a sensible strategy.
  
Without specifying any constraints, a containerized process will be able to see and use all the hardware on the host. If that's not what you want there are several ways to limit the CPU which a container can use in Docker:

  - CPU Share Constraint: `—cpu-shares`  
  ([Doc](https://docs.docker.com/engine/reference/run/#cpu-share-constraint)) This rations the CPU according to the proportions you choose, but only when the system is busy. For example you can have three containers with shares of 1024/512/512, but those limits are only applied when necessary. When there is headroom your containerized process can use left-over CPU time.

  - CPU Period/Quota Constraint: `—cpus` (or `—cpu-period` with `—cpu-quota`)  
  ([Doc](https://docs.docker.com/engine/reference/run/#cpu-period-constraint)) This uses the Linux Completely Fair Scheduler to limit the container's CPU usage. This means that the container will be limited even when the machine is lightly loaded but your workload may still be spread across all the CPUs on the host.
  
  - CPUSet Constraint: `—cpuset-cpus`  
  ([Doc](https://docs.docker.com/engine/reference/run/#cpuset-constraint)) Unlike the two previous constraints, this pins the containerized process to specific CPUs. Your process may have to share those CPUs, but will obviously not be allowed to use spare capacity on any others.

You can mix these options together, too - eg 50% of time on cores 0 through 8. It's well documented by Docker which will link you through to the kernel docs for more info. What we care about now is how big our thread pools will be.

#### Host OS

```shell
$ echo 'Runtime.getRuntime().availableProcessors()' | jdk-10/bin/jshell -q
jshell> Runtime.getRuntime().availableProcessors()
$1 ==> 72
```

#### CPU Shares

```shell
$ echo 'Runtime.getRuntime().availableProcessors()' | docker run --cpu-shares 36864 -i jdk10
jshell> Runtime.getRuntime().availableProcessors()
$1 ==> 36
```

**NB 1** This is based on the relationship 1 CPU = 1024 'shares'. `36864 = 36×1024`.  
**NB 2** That said, 1024 is the default setting, so specifying 1024 here will cause the JVM to report 72 processors. This seems like a gotcha waiting to happen.  
**NB 3** CPU shares are hard for the JVM to work with as the arg you specify is interpreted relative to all the other containers on the system and doesn't mean much by itself.

#### CPU Period/Quota

```shell
$ echo 'Runtime.getRuntime().availableProcessors()' | docker run --cpus 36 -i jdk10
jshell> Runtime.getRuntime().availableProcessors()
$1 ==> 36
```

#### CPU Sets

```shell
$ echo 'Runtime.getRuntime().availableProcessors()' | docker run --cpuset-cpus 0-35 -i jdk10
jshell> Runtime.getRuntime().availableProcessors()
$1 ==> 36
```

The calculation the JVM uses is: `min(cpuset-cpus, cpu-shares/1024, cpus)` rounded up to the next whole number.

It seems logical to use **CPU Sets** when you have enough cores as it should reduce context-switches and increase CPU cache coherency. Or maybe it would be OK to have more threads, if they spend a lot of their time parked and waiting on an interrupt... Contrary to the warnings in the JavaDoc I have not seen any occasions where `.availableProcessors()` changes its result over time within a single JVM.

### Memory

The amount of memory that the JVM will try to allocate and make available is either set explicitly or chosen for you by a process known as *Ergonomics*. The docs state that on a 'server-class' machine (>2 processors, >2Gb RAM) the JVM will run in 'server' mode and max heap size will be set by ergonomics to 1/4 physical memory. In fact in 64bit JVMs there is no alternative to 'server' mode. Additionally the heap size chosen by ergonomics is limited to around 32Gb - if you want more than that you have to ask for it with `-Xmx`.

For example, on my test server (256Gb RAM):

```shell
$ jdk-10+23/bin/java -XX:+PrintFlagsFinal -version | grep MaxHeapSize
   size_t MaxHeapSize                              = 32178700288                              {product} {ergonomic}

```

It seems that ergonomics isn't designed for such large machines. Anyway, let's see what ergonomics makes of a memory-limited container environment:

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


### More Ergonomics

Ergonomics also tunes threadpool sizes used by G1GC, which are reported by the arg `-XX:+PrintFlagsFinal` as `ConcGCThreads` and `ParallelGCThreads` as [described on the OTN](http://www.oracle.com/technetwork/articles/java/g1gc-1984535.html). These are selected by the same means that `Runtime.availableProcessors()` is described above:

#### Host OS

```shell
$ jdk-10/bin/java -XX:+PrintFlagsFinal -version | grep -i GCThreads
     uint ConcGCThreads                            = 12                                       {product} {ergonomic}
     uint ParallelGCThreads                        = 48                                       {product} {default}
```

#### CPU-limited Container


```shell
$ docker run -it --cpus 36  --entrypoint bash jdk10
root@6a94863c54df:/# /java/jdk-10/bin/java -XX:+PrintFlagsFinal -version | grep -i GCThreads
     uint ConcGCThreads                            = 6                                        {product} {ergonomic}
     uint ParallelGCThreads                        = 25                                       {product} {default}
```

## Summary

In JDK10 it does seem that applying CPU and memory limits to your containerized JVMs will be straightforward. The JVM will detect hardware capability of the host correctly, tune itself appropriately and make a good representation of the available capacity to your application.

Thanks to [@msgodf](https://twitter.com/msgodf) and Bob Vandette for help with this post.
