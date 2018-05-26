---
layout:     post
title:      Multi-Stage Docker build with jlink
date:       2018-05-25 10:00:00
summary:    Smaller images for faster startup!
tags:
- java
- jlink
- docker
- fnproject
- draft
---

As I have [written about before](https://mjg123.github.io/tags/#fnproject), I work on [Fn Project](https://fnproject.io) - a serverless compute platform that uses container images to define functions. Unlike some other FaaS platforms, Fn is not prescriptive about supported runtimes - [anything that can run in a container can be a function](https://medium.com/oracledevs/containers-vs-functions-51c879216b97). This means that Fn's users can benefit by improving how they create images.

The best advice I can give across all platforms - no matter what kind of thing you are putting in the function - is to _make your container images as small as possible_.

Small images help in lots of ways. The most obvious is that it takes time to move data over a network. It's easy to create images which are hundreds of megabytes in size, but 100mb takes a whole second to copy over an uncontended gigabit ethernet, and can easily take more. Large images can take longer to build. I have also observed that larger images take longer to start up, which you _do not want_ in a serverless platform.

This post will detail a couple of ways you can make smaller images for your Java applications. We'll start with a naive approach then make the image over 90% smaller using [jlink](https://docs.oracle.com/javase/9/tools/jlink.htm#JSWOR-GUID-CECAC52B-CFEE-46CB-8166-F17A8E9280E9), [Portola](https://wiki.openjdk.java.net/display/portola/Main) and ["multi-stage" Docker builds](https://docs.docker.com/develop/develop-images/multistage-build/).

## Some Simple Java 

A class:

```java
public class HelloWorld {
  public static void main(String... args){
    System.out.println("Hello from a container!");
  }
}
```

And a straightforward Dockerfile:

```
FROM oraclelinux:7-slim
ADD openjdk-10_linux-x64_bin.tar.gz /opt/jdk
ENV PATH=$PATH:/opt/jdk/jdk-10/bin
ADD HelloWorld.class /
CMD ["java", "-showversion", "HelloWorld"]
```

Of course there are other ways to make this into a container image but this is a reasonable first attempt. And it works:

```
⇒ docker build -t simple-java .
Sending build context to Docker daemon  204.9MB
Step 1/5 : FROM oraclelinux:7-slim
 ---> 9870bebfb1d5
Step 2/5 : ADD openjdk-10_linux-x64_bin.tar.gz /opt/jdk
 ---> Using cache
 ---> 275fbb9eca10
Step 3/5 : ENV PATH $PATH:/opt/jdk/jdk-10/bin
 ---> Using cache
 ---> 9ea350531bfd
Step 4/5 : ADD HelloWorld.class /
 ---> Using cache
 ---> 758fae6f027d
Step 5/5 : CMD java -showversion HelloWorld
 ---> Using cache
 ---> 714108a12dca
Successfully built 714108a12dca
```

```
⇒ docker run simple-java
openjdk version "10" 2018-03-20
OpenJDK Runtime Environment 18.3 (build 10+46)
OpenJDK 64-Bit Server VM 18.3 (build 10+46, mixed mode)
Hello from a container!
```

But - here's the problem...

```
⇒ docker image ls | grep simple-java
simple-java  latest  714108a12dca   2 minutes ago   461MB
```

This image is 461mb! Nearly half a gigabyte! It's crazy. `oraclelinux:7-slim` is 118mb, and the JDK added in step 2/5 is 329mb. Both of which are unnecessarily large.

## Multi-Stage Docker Builds

Many languages (including Java) need to be compiled before they can be run. Often the tools you need for compilation are not needed at runtime. For example you might need maven to build your project, but you don't need maven when you run it.

Multi-stage Docker builds let you have many named stages, each of which can be used as a basis for later stages. Commonly you would have a _build_ stage with compiler and build tools in it, and a _run_ stage which only needs the files you want at runtime. For this example I'll use `jlink` in a _build_ stage to create a small JRE which I can copy into a _run_ stage.

## jlink

The biggest target for reduction in our original image is the JDK, at 329mb.  Lets see how to reduce this with a multi-stage build.

A multi-stage Dockerfile looks like multiple Dockerfiles in the same file, and you can copy a subset of file from one stage to another. The final stage is the only one which becomes the image, so it won't contain contents of the previous stages unless you copy them explicitly. This is our multi-stage build which uses `jlink` in the first stage:

```
FROM oraclelinux:7-slim AS build
ADD openjdk-10_linux-x64_bin.tar.gz /opt/jdk
ENV PATH=$PATH:/opt/jdk/jdk-10/bin
RUN ["jlink", "--compress=2", \
     "--module-path", "/opt/jdk/jdk-11/jmods", \
	 "--add-modules", "java.base", \
	 "--output", "/linked"]

FROM oraclelinux:7-slim
COPY --from=build /linked /opt/jdk/
ENV PATH=$PATH:/opt/jdk/bin
ADD HelloWorld.class /
CMD ["java", "-showversion", "HelloWorld"]
```

```
⇒ docker run jlink
openjdk version "10" 2018-03-20
OpenJDK Runtime Environment 18.3 (build 10+46)
OpenJDK 64-Bit Server VM 18.3 (build 10+46, mixed mode)
Hello from a container!
```

And lets see the size reduction:

```
⇒ docker images -a | grep 7b17ae206906
jlink   latest   7b17ae206906    About a minute ago   152MB
```

152mb is already a significant improvement over the 461 we had before. We could stop here, having saved 2/3 of the size of the image. But we won't.

### A Multi-Stage Dockerfile in Detail

```
FROM oraclelinux:7-slim AS build
```

We're using Oracle Linux as before, and we name a _stage_ by using the `AS` directive.

```
ADD openjdk-10_linux-x64_bin.tar.gz /opt/jdk
ENV PATH=$PATH:/opt/jdk/jdk-10/bin
```

This unzips the whole JDK into `/opt/jdk` (which is still 329mb) and add it to our `$PATH`.

```
RUN ["jlink", "--compress=2", \
     "--module-path", "/opt/jdk/jdk-11/jmods", \
     "--add-modules", "java.base", \
     "--output", "/linked"]
```

The `RUN` directive uses `jlink` to create a small JDK containing only the `java.base` module. I know up-front that `java.base` is all we need here, but if you have a more complex build then use [`jdeps`](https://docs.oracle.com/javase/9/tools/jdeps.htm) to work out what modules you need. The point here is that there will be the whole JDK in `/opt/jdk` and a small version of the it in `/linked`.

```
FROM oraclelinux:7-slim
```

It looks like we're starting again! It's a new stage. There's no need to use the same base image for each stage, but it's convenient for me here.

```
COPY --from=build /linked /opt/jdk/
ENV PATH=$PATH:/opt/jdk/bin
```

This is the multi-stage magic. We can copy some files from the previous stage using `COPY --from=<previous-stage>`

```
ADD HelloWorld.class /
CMD ["java", "-showversion", "HelloWorld"]
```

As before we need to add the code we want to run, and set the `CMD`.

Copying only what we need from one stage to the next is one way that we can use multi-stage builds to reduce image size. The other way I will reduce this image's size is by focussing on the base image we're using.

## Portola

As Java programmers we're used to relying on the JVM to abstract us away from the OS, so all we really need from a base image is *just enough* OS to run the JVM.

As we saw, the Oracle Linux 7-slim base image is over 100mb, but we can use [Alpine Linux](https://alpinelinux.org/) as our base image instead. Alpine is a really tiny Linux distribution (4.15mb!), but it's powerful enough to run a JVM.

Because Alpine uses [musl-libc](https://www.musl-libc.org/) instead of glibc we can't use a regular JVM but in the JDK11 Early-Access builds there is a build for MUSL. This is a port of the JVM which runs on musl-bases Linuxes, the project which produces it is called [Project Portola](http://openjdk.java.net/projects/portola/).

So, the multi-stage Dockerfile now looks like this:

```
FROM alpine:latest AS build
ADD openjdk-11-ea+4_linux-x64-musl_bin.tar.gz /opt/jdk
ENV PATH=$PATH:/opt/jdk/jdk-11/bin
RUN ["jlink", "--compress=2", "--module-path", "/opt/jdk/jdk-11/jmods", "--add-modules", "java.base", "--output", "/linked"]

FROM alpine:latest
COPY --from=build /linked /opt/jdk/
ENV PATH=$PATH:/opt/jdk/bin
ADD HelloWorld.class /
CMD ["java", "-showversion", "HelloWorld"]
```

```
⇒ docker run portola-jlink
openjdk version "11-ea" 2018-09-18
OpenJDK Runtime Environment 18.9 (build 11-ea+4)
OpenJDK 64-Bit Server VM 18.9 (build 11-ea+4, mixed mode)
Hello from a container!
```

So, the behaviour is exactly the same, but check this out:

```
⇒ docker images -a | grep 3007ac31eedb
portola-jlink latest  3007ac31eedb  2 minutes ago   37.9MB
```

This image is less than 38mb! More than 90% file size reduction since the beginning but the resulting behaviour is exactly the same. How wonderful!


### A Quick Word About Portola

Portola is a JDK11-EA build at the moment. The decision about whether to make it a supported build for 11 onwards is going to be made soon, and will be highly driven by the expected demand for it. So if you are interested in using it, please try it out and talk to us!


# Summary

As promised, we cut more than 90% of the size of a naive Docker image for a Java application. If you try this out and it helps you I'd love to know about it! Questions, comments etc please find me on Twitter.
