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

I can't really agree with his title though - lots and lots of people *do* put JVM workloads in containers (spoiler: I have done so on my last several projects - we even [tune for it](https://github.com/fnproject/fdk-java/blob/master/runtime/Dockerfile-jdk9#L6-L18) in [Fn Project](http://fnproject.io)). As Jörg points out, when JDK10 is released support will be even better. In this post we'll see how the next release of the JDK will work in containers.

I'll be using the Docker to run the latest nightly build of JDK10 - follow the `browser_download_url` for your architecture from [here](https://github.com/AdoptOpenJDK/openjdk10-nightly/blob/master/latest_nightly.json) ([Dockerfile](https://gist.github.com/mjg123/cbdee8a9ecba76ec19853d0ac0269d3d)). And I'll be using a baremetal instance on [Oracle Cloud Infrastructure](https://cloud.oracle.com/en_US/infrastructure/compute) which comes with 72 cores and 256Gb of RAM. Just the kind of place to worry about how best to run lots of things concurrently 😉

There are two places where the JVM's view of what hardware is available is important: user code and the JVM ergonomics.

## User code

### CPUs

Many apps will use [`Runtime.getRuntime().availableProcessors()`](http://download.java.net/java/jdk10/docs/api/java/lang/Runtime.html#availableProcessors()) to size thread pools, for example:

  - [core.async](https://github.com/clojure/core.async/blob/d81acd/src/main/clojure/clojure/core/async/impl/concurrent.clj#L28-L30)
  - [ElasticSearch](https://github.com/clojure/core.async/blob/d81acd/src/main/clojure/clojure/core/async/impl/concurrent.clj#L28-L30)
  - [Netty](https://github.com/netty/netty/blob/98beb777f81f092aa0fddf49ed08b426b2c72f01/common/src/main/java/io/netty/util/NettyRuntime.java#L69)
  
  ... and so on. It's a sensible strategy.
  
There are two ways to limit the CPU which a container can use, **CPU shares** and **CPU sets**. **Shares** will give you a proportion of the available CPU time across all cores, whereas **sets** will pin your container process to a certain subset of available cores. It's [well documented](https://docs.docker.com/engine/reference/run/#cpu-period-constraint).

So, how big should we make our threadpools?

#### Host OS

```shell
$ echo 'java.lang.Runtime.getRuntime().availableProcessors()' | jdk-10+23/bin/jshell -q
jshell> java.lang.Runtime.getRuntime().availableProcessors()
$1 ==> 72
```

#### CPU Shares

```shell
$ echo 'java.lang.Runtime.getRuntime().availableProcessors()' | docker run --cpus 36 -i jdk10
jshell> java.lang.Runtime.getRuntime().availableProcessors()
$1 ==> 72
```

#### CPU Sets

```shell
$ echo 'java.lang.Runtime.getRuntime().availableProcessors()' | docker run --cpuset-cpus 0-35 -i jdk10
jshell> java.lang.Runtime.getRuntime().availableProcessors()
$1 ==> 36
```

It seems logical to use **CPU Sets** when you have enough cores as it should reduce context-switches and increase CPU cache coherency. Or maybe it would be OK to have more threads, if they spend a lot of their time parked and waiting on an interrupt... Naturally you would profile your perf-sensitive app to check that intuition right. Right?!

### Memory

The amount of memory that the JVM will try to allocate and make available is either set explicitly or chosen for you by a process known as *Ergonomics*. The docs state that on a large machine (>2 processors, >2Gb RAM) the max heap size will be set by ergonomics to 1/4 physical memory. I haven't found that this is true on very large machines.

For example, on my test server (72 cores, 256Gb RAM):

```shell
$ jdk-10+23/bin/java -XX:+PrintFlagsFinal -version | grep MaxHeapSize
   size_t MaxHeapSize                              = 32178700288                              {product} {ergonomic}

```

This is around 32Gb, although I can specify more with `-Xmx` if I need, it seems that ergonomics isn't designed to cope with such large machines.

So, let's see what ergonomics makes of a memory-limited container environment:

```
$ docker run -it -m512M  --entrypoint bash jdk10
root@7378a1f0a2a9:/# /java/jdk-10+23/bin/java  -XX:+PrintFlagsFinal -version | grep MaxHeapSize
   size_t MaxHeapSize                              = 32178700288                              {product} {ergonomic}
```

Oh no! That's still 32Gb. And what does it look like from inside the JVM?

```shell
$ docker run -it -m512M  jdk10
jshell> Runtime.getRuntime().maxMemory()
$1 ==> 32178700288
```

Oh no, again! Perhaps, though, the docker constraint is not actually being applied...? Let's allocate some memory - a billion bytes ought to do it...

```
jshell> new byte[1_000_000_000]
|  State engine terminated.
|  Restore definitions with: /reload -restore
$2 ==> 
```

... ??? ... try again ...

```
shell> new byte[1_000_000_000]
```

the process has dies and we are back at the host shell.

```
$ echo $?
137
```

`jshell` is terminated, the exit code 137 indicates that it was killed. `dmseg` says:

```
[76062.885768] Task in /docker/a3e12dff9d68bc13bde666d06a99315faa310f2d6b35ee101e180440af2f3315 killed as a result of limit of /docker/a3e12dff9d68bc13bde666d06a99315faa310f2d6b35ee101e180440af2f3315
[76062.885775] memory: usage 524288kB, limit 524288kB, failcnt 212
[76062.885778] memory+swap: usage 0kB, limit 9007199254740988kB, failcnt 0
[76062.885779] kmem: usage 0kB, limit 9007199254740988kB, failcnt 0
[76062.885781] Memory cgroup stats for /docker/a3e12dff9d68bc13bde666d06a99315faa310f2d6b35ee101e180440af2f3315: cache:72KB rss:524216KB rss_huge:344064KB mapped_file:0KB dirty:0KB writeback:0KB inactive_anon:20KB active_anon:512104KB inactive_file:12KB active_file:16KB unevictable:0KB
[76062.885809] [ pid ]   uid  tgid total_vm      rss nr_ptes nr_pmds swapents oom_score_adj name
[76062.885863] [29816]     0 29816 11012521    94332     502      16        0             0 jshell
[76062.885867] Memory cgroup out of memory: Kill process 29816 (jshell) score 723 or sacrifice child
[76062.992129] Killed process 29816 (jshell) total-vm:44050084kB, anon-rss:348936kB, file-rss:28392kB
```

So we became the latest victim of the OOM killer.



