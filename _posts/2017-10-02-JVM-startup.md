---
layout:     post
title:      Fast JVM startup with JDK 9
date:       2017-10-02 12:31:19
summary:    Tuning for short-lived JVM processes
tags:
- jvm
---

Recently I've been working on a project where the execution time of short-lived JVM workloads is a critical factor in overall system performance. This isn't a particularly common type of workload for the JVM, which is usually long-lived apps that allow HotSpot profiling and optimising to do a really amazing job of speeding your code up.

In our processes only a small handful of methods were run often enough to be compiled; 99% of everything was run in the JVM's bytecode-interpreted mode. Instead of focusing on runtime optimisations I decided to attack the startup time. I was lucky to get a lot of help from the JVM team here at Oracle, who recommended two techniques:

  * Class Data Sharing (CDS), available since Java 5
  * Ahead-Of-Time compilation (AOT), new in Java 9

Both made a noteable difference, but the docs are a bit terse and there's a *lot* of flags which you can use. Read on for some actual examples of how to use these features to speed your JVM startup. Let's get to it:


## A Simple App

I've installed Java 9:

```shell
⇒ java -version
java version "9"
Java(TM) SE Runtime Environment (build 9+181)
Java HotSpot(TM) 64-Bit Server VM (build 9+181, mixed mode)
```

And I wrote a *really* simple Java app:

```java
public class HelloJava {
    public static void main(String... args){
        System.out.println("Hello Java!");
    }
}
```

Then I compiled with `javac HelloJava.java` and ran it:

```shell
⇒ java HelloJava
Hello Java!
```

Yay!

## Baseline

we can measure performance with shell builtin `time`:

```shell
⇒ time java HelloJava
Hello Java!
java HelloJava  0.11s user 0.02s system 117% cpu 0.113 total

```

but it's better to use `perf` ([linux only](https://perf.wiki.kernel.org/index.php/Main_Page)) which can run the command several times and gather a lot of different statistics. I've only selected cpu-clock but perf is a real [Swiss-Army knife of process measurement](http://www.brendangregg.com/perf.html). By default `perf` needs to be run as root, and I've chosen to run 50 times to get a decent average without taking too long:

```shell
⇒ sudo perf stat -e cpu-clock -r50 \
    java HelloJava
Hello Java! # printed 50x

 Performance counter stats for 'java HelloJava' (50 runs):

        139.614197      cpu-clock (msec)          #    1.081 CPUs utilized            ( +-  0.89% )

       0.129159131 seconds time elapsed                                          ( +-  1.63% )

```

This shows that a JVM startup, print of a single line of text, shutdown takes around 129ms.

## CDS

[Class Data Sharing](https://docs.oracle.com/javase/9/vm/class-data-sharing.htm#JSJVM-GUID-EC975B2E-B4AB-45B4-B91F-51C3A264D0CE) is a JVM feature which is intended to remove the fixed cost of loading core classes at startup. It also allows some cached files to be shared between JVMs which helps if you're running a lot of them. This is cool because it can be made to work even if the JVMs are containerised.

CDS is easy to set up, just run with `-Xshare:dump` to generate the cache.

```
⇒ java -Xshare:dump
Allocated shared space: 50577408 bytes at 0x0000000800000000
Loading classes to share ...
Loading classes to share: done.
Rewriting and linking classes ...
Rewriting and linking classes: done
Number of classes 1192
    instance classes   =  1178
    obj array classes  =     6
    type array classes =     8
Updating ConstMethods ... done. 
Removing unshareable information ... done. 
ro space:   5355536 [ 30.5% of total] out of  10485760 bytes [ 51.1% used] at 0x0000000800000000
rw space:   5627928 [ 32.1% of total] out of  10485760 bytes [ 53.7% used] at 0x0000000800a00000
md space:    136544 [  0.8% of total] out of   4194304 bytes [  3.3% used] at 0x0000000801400000
mc space:     34053 [  0.2% of total] out of    122880 bytes [ 27.7% used] at 0x0000000801800000
st space:     12288 [  0.1% of total] out of     12288 bytes [100.0% used] at 0x00000007bff00000
od space:   6372368 [ 36.3% of total] out of  20971520 bytes [ 30.4% used] at 0x000000080181e000
total   :  17538717 [100.0% of total] out of  46272512 bytes [ 37.9% used]
```

This creates the file `$JAVA_HOME/lib/server/classes.jsa`.


Lets re-run the perf check with CDS on:

```shell
⇒ sudo perf stat -e cpu-clock -r50 \
    java -Xshare:on HelloJava
... 50 lines of output elided ...

 Performance counter stats for 'java -Xshare:on HelloJava' (50 runs):

        105.569185      cpu-clock (msec)          #    1.093 CPUs utilized            ( +-  0.75% )

       0.096572606 seconds time elapsed                                          ( +-  1.49% )
```

So this is a pretty big improvement in runtime, 30ms right off the bat. If we had a larger app, it would be possible to use AppCDS to preload more than just java core classes (but that's a topic for another post).  The other big advantage of CDS is that the cache file is `mmap`-ed read-only, so can be shared between multiple JVMs. This has a nice consequence if you're running JVMs in containers, that the shared-page usage only counts towards one of the cgroup limits (although *which one* seems to be [difficult to predict](https://github.com/torvalds/linux/blob/master/Documentation/cgroup-v1/memory.txt#L195-L199)).

We know that the CDS cache has been used, because `Xshare:on` will fail if the cache isn't found. We can also check where the classes are loaded from with the argument `-Xlog:class+load=info` which was previously known as `-XX:+TraceClassLoading`.

With CDS we see output like this:

```shell
⇒ java -Xshare:on -Xlog:class+load=info HelloJava
[0.004s][info][class,load] opened: /home/mjg/tools/jdk-9/lib/modules
[0.014s][info][class,load] java.lang.Object source: shared objects file
[0.014s][info][class,load] java.io.Serializable source: shared objects file
[0.014s][info][class,load] java.lang.Comparable source: shared objects file
[0.015s][info][class,load] java.lang.CharSequence source: shared objects file
...etc...
```

and without CDS:

```shell
⇒ java -Xlog:class+load=info HelloJava
[0.003s][info][class,load] opened: /home/mjg/tools/jdk-9/lib/modules
[0.014s][info][class,load] java.lang.Object source: jrt:/java.base
[0.014s][info][class,load] java.io.Serializable source: jrt:/java.base
[0.014s][info][class,load] java.lang.Comparable source: jrt:/java.base
[0.015s][info][class,load] java.lang.CharSequence source: jrt:/java.base
...etc...
```

**NB 1** apparently the default behaviour is `-Xshare:off` when the JVM is in server mode, though [this may change in a future update](https://bugs.openjdk.java.net/browse/JDK-8188105), so I explicitly set it to `:on` in the above tests.  According to [this issue](https://bugs.openjdk.java.net/browse/JDK-8188109) it is safest to select `:auto` if you're using CDS in real life.


**NB 2** Up until jdk8 CDS used to only work with Serial GC but this restricition is lifted in jdk9. I confirmed that the effect is similar with Serial, Parallel or G1 GC.


## AOT

[AOT is an experimental new feature in jdk9](http://openjdk.java.net/jeps/295) for linux-x86 JVMs. Where CDS does some parts of classloading of core classes in advance, AOT actually compiles bytecode to native code (an ELF-format shared-object file) in advance, and can be applied to any bytecode.

To use aot we need to use a new tool from `$JAVA_HOME/bin` called `jaotc`, and we need to decide *what* to AOT compile. The simplest decision to make is to compile the whole `java.base` module:

```shell
⇒ jaotc --output java_base.so --module java.base --info -J-Xmx4g
Compiling java_base...
5747 classes found (834 ms)
54546 methods total, 50909 methods to compile (3424 ms)
Compiling with 2 threads
...................................................................................................................................................................................................Error: Failed compilation: sun.reflect.misc.Trampoline.invoke(Ljava/lang/reflect/Method;Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;: org.graalvm.compiler.java.BytecodeParser$BytecodeParserError: java.lang.Error: Trampoline must not be defined by the bootstrap classloader
        at parsing java.base@9/sun.reflect.misc.Trampoline.invoke(MethodUtil.java:70)
Error: Failed compilation: sun.reflect.misc.Trampoline.<clinit>()V: org.graalvm.compiler.java.BytecodeParser$BytecodeParserError: java.lang.NoClassDefFoundError: Could not initialize class sun.reflect.misc.Trampoline
        at parsing java.base@9/sun.reflect.misc.Trampoline.<clinit>(MethodUtil.java:50)
...........................................................................................................................................................................................................................................................................................................................
50907 methods compiled, 2 methods failed (452365 ms)
Parsing compiled code (2252 ms)
Processing metadata (32259 ms)
Preparing stubs binary (1 ms)
Preparing compiled binary (493 ms)
Creating binary: java_base.o (13054 ms)
Creating shared library: java_base.so (7763 ms)
Total time: 526783 ms

```

As you can see from the last line, this takes quite a long time. The errors about `Trampoline` seem to be normal. We'll see how to get rid of them in a bit. The so file is reusable so long as we use the same JVM at runtime.  Compilation is done by [Graal](https://github.com/graalvm/graal) which is an awesome and new bytecode compiler written in Java.

The shared object file created is quite large:

```shell
⇒ ls -lh ./java_base.so
-rw-r--r-- 1 mjg mjg 321M Sep 27 10:41 ./java_base.so
```

You may suspect that we've spent longer and generated more than we actually need to, and you'd be right. We'll dig into that later. For now, use the AOT file like this:

```shell
⇒ java -XX:AOTLibrary=./java_base.so HelloJava 
Hello Java!
```

So, drum-roll, what's the effect on performance?

```shell
⇒ sudo perf stat -e cpu-clock -r50 \
    java -XX:AOTLibrary=./java_base.so  HelloJava
...50 lines elided...

 Performance counter stats for 'java -XX:AOTLibrary=./java_base.so HelloJava' (50 runs):

        131.587104      cpu-clock (msec)          #    0.913 CPUs utilized            ( +-  0.74% )

       0.144118938 seconds time elapsed                                          ( +-  0.88% )
```

Ah crap! We've gone and made it worse. Not only did we compile too much, we're now loading too much and it's slowed down the JVM start time. Actually the `cpu-clock` time is about the same with or without AOT here, so the extra time must be taken up off-cpu... It's caused by the I/O to load the 321mb cache file.

### A few asides about AOT

#### Aside 1: Where can I use it?

AOT is an "experimental" feature, only currently supported on 64-bit JVMs on linux-x86 (all JVMs are 64-bit from 9 onwards). This is what most people's prod environments look like, in my experience, but the JVM is widely used in plenty of other places too. There is [an RFE for AOT support in Windows and MacOS](https://bugs.openjdk.java.net/browse/JDK-8172670) which seems to be progressing well, so we might get to see in a near-future release.

#### Aside 2: Tiered compilation

Is is possible to AOT-compile your bytecode such that it is completely static which should lead to the fastest startup and initial execution speed. Or you can have profiling code added so that the bytecode can be optimised by the existing HotSpot compiler as normal (aka "tiered" compiliation). Tiered AOT should lead to faster code at startup *and* no loss of peak performance.

Whether or not the profiling information is collected is controlled by the argument `--compile-for-tiered`, which defaults to off. For short-lived JVMs this is the right choice, but if you expect to reach peak performance in long-running JVMs taking advantage of HotSpot's optimisations, you should use `--compile-for-tiered`.

#### Aside 3: GC and other JVM tuning options

`jaotc` takes arguments of the form `-J<arg>` which are passed down to the JVM doing the compilation with Graal. You might have noticed I used `-J-Xmx4g` above. Some arguments *need* to be the same between this and the runtime JVM, notably the choice of GC. You can check whether your runtime is able to use the AOT cache by using `-XX:+UseAOTStrictLoading`, which needs `-XX:+UnlockDiagnosticVMOptions`. For example, remembering that `jaotc` used G1GC (jdk9 default), if I manually specify `-XX:+UseParallelGC`, we see:

```shell
⇒ java -XX:AOTLibrary=./java_base.so \
       -XX:+UnlockDiagnosticVMOptions \
       -XX:+UseAOTStrictLoading \
       -XX:+UseParallelGC \
       HelloJava
⇒ echo $?
1
```

Silent failure, but the error code is instructive.


## More selective AOT ##

The results for AOT so far are a bit disappointing, to put it mildly. What can we do to improve matters? Well, [the docs for `jaotc`](http://openjdk.java.net/jeps/295) are quite brief, but they point in some useful directions. For example, it is possible to AOT compile a whole module (as we have done), or a directory, or a jar, or an individual class. There's even a `--compile-comands` which looks like it might be able to specify what to compile down to the individual method. Let's see.

We'd like to use `--compile-commands`, so we need to actually know which methods we need to compile. Time for some more JVM flags!

```shell
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+LogTouchedMethods \
     -XX:+PrintTouchedMethodsAtExit \
     HelloJava > touched_methods
```

The file `touched_methods` now has a long list of the almost 2000 methods we used in that short time. It also has the stdout from our process (ie `Hello Java!`) and a header line. It's also not quite in the format that `--compile-commands` expects, as the classes are written like `java/lang/Sytem` but `jaotc` wants `java.lang.System`. Oh, and we need to prefix each line with `compileOnly`.

I was also given a tip-off by the Java team that the a couple of methods from `jdk.internal.module.SystemModules` should be excluded as they don't benefit from AOT and are large. I verified that removing them saves around 9mb from the .so file and has a positive effect on performance so I'd recommend you do the same.

This is nothing we can't fix manually but it would be nice if there was some tooling for doing this. There's [an RFE for making `jaotc` more flexible](https://bugs.openjdk.java.net/browse/JDK-8184308), which should alleviate some of the work.

```shell
⇒ grep -v 'Hello Java!' touched_methods | \
    grep -v '^#' | \
    grep -v jdk/internal/module/SystemModules.hashes | \
    grep -v jdk/internal/module/SystemModules.descriptors | \
    sed -e 's/^/compileOnly /' | \
    java Convert > touched.aotcfg
⇒ head -2 touched.aotcfg
compileOnly java.lang.System.setErr0(Ljava/io/PrintStream;)V
compileOnly jdk.internal.module.Builder.packages(Ljava/util/Set;)Ljdk/internal/module/Builder;
```

`Convert.java` is a [little tool for fixing the method name](https://gist.githubusercontent.com/mjg123/d06fd39ce1aedb3032edd463e814cdd1/raw/c9497e6a1da5a656d181986ac094ba4d95a88dfa/Convert.java) formatting. Now the file `touched.aotcfg` is in the right format.

Lets try recreating the AOT cache with only the touched methods in the cache. We specify the `java.base` module and our class file as sources:

```shell
⇒ jaotc --output touched_methods.so \
        --compile-commands touched.aotcfg \
        --module java.base \
        --class-name HelloJava.class \
        --info
Compiling touched_methods...
5748 classes found (594 ms)
54548 methods total, 1874 methods to compile (840 ms)
Compiling with 2 threads
...................
1874 methods compiled, 0 methods failed (24642 ms)
Parsing compiled code (80 ms)
Processing metadata (707 ms)
Preparing stubs binary (0 ms)
Preparing compiled binary (6 ms)
Creating binary: touched_methods.o (205 ms)
Creating shared library: touched_methods.so (171 ms)
Total time: 28753 ms
```

So that was much quicker, and the .so file is correspondingly smaller. It is possible to compile several different AOT caches and use many at once, but I'm happy with putting everything into a single .so file for now:

```shell
⇒ ls -lh touched_methods.so 
-rw-r--r-- 1 mjg mjg 11M Sep 27 13:12 touched_methods.so
```

That was nearly 20x faster, and the output is about 30x smaller. What we're actually interested in, though, is the time it takes to run our app:

```shell
⇒ sudo perf stat -e cpu-clock -r50 \
    java -XX:AOTLibrary=./touched_methods.so  HelloJava

 Performance counter stats for 'java -XX:AOTLibrary=./touched_methods.so HelloJava' (50 runs):

         90.295446      cpu-clock (msec)          #    0.941 CPUs utilized            ( +-  0.46% )

       0.095916063 seconds time elapsed                                          ( +-  0.51% )
```

96ms is pretty good, about the same improvement as we got with CDS.

## ¿Por qué no los dos?

I wasn't sure if CDS and AOT would interfere with each other, so I tried both and it seems to have a good result.

```shell
⇒ sudo perf stat -e cpu-clock -r50 \
    java -XX:AOTLibrary=./touched_methods.so -Xshare:on HelloJava

 Performance counter stats for 'java -XX:AOTLibrary=./touched_methods.so -Xshare:on HelloJava' (50 runs):

         60.141333      cpu-clock (msec)          #    0.877 CPUs utilized            ( +-  0.60% )

       0.068579396 seconds time elapsed                                          ( +-  1.18% )
```

## Summarizing

Compared to what the beginning this is a significant improvement:


![Bar chart showing improved JVM execution times with CDS and AOT](https://gist.githubusercontent.com/mjg123/d06fd39ce1aedb3032edd463e814cdd1/raw/c9497e6a1da5a656d181986ac094ba4d95a88dfa/barchart.png)


The saving in CPU-time is significant too, which will yield benefits when there are multiple JVMs running on a single host. The CDS and AOT caches can be almost completely shared between processes.

## Acknowledgements

As I said above, I had help from the JVM team in figuring all this out. In particular I'd like to thank Ioi Lam, Claes Redestad, Karen Kinnear, Mary Urillo, Alan Bateman, Eric Caspole, Bob Vandette, Jiangli Zhou and Vladimir Kozlov for their help, technical advice and for proof-reading.

If you want to discuss more, grab me on Twitter or head to [the Reddit post](https://www.reddit.com/r/java/comments/748w96/fast_jvm_startup_with_jdk_9_and_jdk5/) for this article. Thanks.

## Next up

Several topics have come out of this. I'm going to look into:

  * Applying AOT realistic workload.  Whatever you're doing you will have to load classes from `java.base` so what we've seen so far is generally applicable, but it will be interesting to see what effect it has on something bigger than "Hello Java!"
  * AppCDS, "Application CDS" will apply the same improvements to application code does to core Java classes.
  * FlameGraph -ing the JVM. An excellent tool for visualising what is happening inside the JVM itself.
