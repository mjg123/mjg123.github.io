---
layout:     post
title:      Java 11 in Alpine Linux containers
date:       2018-11-05 18:00:00
summary:    how to build small container images at home
tags:
- java
- containers
- draft
---

Now that JDK 11 is fully released with [a raft of excellent new features](https://openjdk.java.net/projects/jdk/11/), many developers are looking to use it in containerized environments. This is an excellent time to do so, with [many new container-friendly features](https://docs.google.com/presentation/d/11VjOwW8MjDqXX9uRx0BEGYrIQtGGcXJJWMxS2q-02nA/edit#slide=id.g3c0528a66b_1_162) being added in each release since JDK 8.

If you are a conscientious image-builder, you will have heard many times the advice to keep 'em small. From [Google](https://cloud.google.com/blog/products/gcp/7-best-practices-for-building-containers), from [me](https://vimeo.com/289497209), from [Red Hat](https://developers.redhat.com/blog/2016/03/09/more-about-docker-images-size/) or from someone else.  The reasons are numerous: 

  - small images are faster to copy around, 
  - they are faster to start, 
  - by removing unneeded things you are reducing the damage that a security exploit could do.
  
This post talks through some of the options and pitfalls in building small JVM container images based on Alpine Linux.

## Alpine, Musl and the JVM

For many use-cases, [Alpine Linux](https://alpinelinux.org/) is the ideal base image to use. First of all it's very small - 4.4mb, which is only a couple of percent of the size of a typical "slim" Linux image.

However, there is something you have to know about Alpine: it does not use **[glibc](https://www.gnu.org/software/libc/)**, instead it uses **[musl](https://www.musl-libc.org/)**. What are glibc and musl? They are programming APIs for the Linux Kernel, doing such things as opening files or network connections. Although they are abstractions over the same underlying thing (the Linux Kernel binary interface), they expose slightly different APIs. So C or C++ code which is compiled against glibc _will not run_ on a musl system, and vice-versa.

Why am I telling you, dear Java programmer, about C and C++ Kernel APIs?  Well, because you use an application written in C++ to do a lot of your work: The JVM!  And how the JVM interacts with the Linux kernel is critical to how it can be used in containers.  If you want to put your Java or Clojure or Kotlin or Scala application in a container, and benefit from Alpine's small size, you need some way to run your JVM on musl!

## Project Portola

So, we know there are some code changes needed to allow the JVM to run on musl-based systems - lucky for us then, that such changes _do exist_ in the repository of [Project Portola](https://openjdk.java.net/projects/portola/). Oracle is publishing Early-Access builds of the Portola code base for the in-development JDK version (currently [JDK 12](https://jdk.java.net/12)) and is gauging the interest in the port to decide how to move forward with it. Given the experimental status of the port there are not yet any GA versions of it. In fact I have not found any publicly available builds of OpenJDK 11 (or earlier) for musl from any vendor.

But, don't worry, there is a way forward.

## Glibc Compatibility for JVM on Alpine

[Sasha Gerrand](https://github.com/sgerrand) maintains a [glibc package for Alpine Linux](https://github.com/sgerrand/alpine-pkg-glibc). If you add this package to an Alpine system, you will be able to run glibc-based applications - including any glibc-based JDK or JRE - WOW!

This is how the [Alpine images](https://github.com/AdoptOpenJDK/openjdk-docker#supported-builds-and-build-types) are produced by [AdoptOpenJDK](https://adoptopenjdk.net/) - they _do not use_ a musl port of the JVM. The total size of Alpine plus the glibc-compatibility layer (and all the tools needed to install it) is 52mb. This is a lot bigger than the base Alpine image, but still a lot smaller than ubuntu, for example.

For the most part you can just take one of the AdoptOpenJDK images and run your application as usual, enjoying the 100+ megabyte image size reduction. The largest component of your image is now likely to be the JDK.  You _could_ stop here, but if you want even smaller images, read on...


## Using jlink to shrink the JDK

You are _highly unlikely_ to be using all of the features that the JDK provides.  I've written about using `jlink` to shrink JDK distributions [before](https://mjg123.github.io/2017/11/07/Java-modules-and-jlink.html), [twice](https://mjg123.github.io/2018/05/26/Multi-Stage-Docker-Build-with-jlink.html) - the TL;DR is that `jlink` can provide a JDK distribution which contains _only_ the Java modules you need. This can save a lot of space.

If you are using Portola-based JDKs then the smaller JDK which `jlink` creates will be fine to run on Alpine as-is, and you can have a Dockerfile like this:

```
FROM alpine:latest as build

# This is the latest Portola JDK distribution at the time of writing
ADD https://download.java.net/java/early_access/alpine/18/binaries/openjdk-12-ea+18_linux-x64-musl_bin.tar.gz /opt/jdk

RUN ["/opt/jdk/jdk-12/jlink", "--compress=2", \
     "--module-path", "/opt/jdk/jdk-12/jmods", \
     "--add-modules", "java.base", \
     "--output", "/jlinked"]


FROM alpine:latest
COPY --from=build /jlinked /opt/jdk/
ADD HelloWorld.class /
CMD ["/opt/jdk/java", "HelloWorld"]
```

However, if you try using `adoptopenjdk/openjdk-11:alpine-slim` as your base image for the first stage then you will notice two things:

  - you won't need to use the `ADD` line - the JDK's already there
  - rather importantly, you will also find that the resulting image _does't work_
  
It will fail with a rather confusing `not found` error. The error is because the `adoptopenjdk/openjdk-11:alpine-slim` JDK builds use the regular glibc-based JVMs, and `jlink` from those will produce another glibc-based JVM. So those JVMs in a plain `alpine:latest` image are gonna fail to start.  You need to _manually_ install the glibc-compatibility package in your final image, just as was done in the base image from AdoptOpenJDK.

The easiest way to do that right now is to copy exactly how AdoptOpenJDK have done it, which you can find in a rather beefy `RUN` command in their [Alpine Dockerfiles](https://github.com/AdoptOpenJDK/openjdk-docker/blob/2baf4481c1a3a70f47a8aae074ec9a4027945638/11/jdk/alpine/Dockerfile.hotspot.releases.slim#L24-L46).  A Dockerfile which you can use as a basis for your own builds from is as follows:

```Dockerfile
FROM adoptopenjdk/openjdk11:alpine-slim AS jlink
RUN ["jlink", "--compress=2", \
     "--module-path", "/opt/java/openjdk/jmods", \
     "--add-modules", "java.base", \   # Maybe you need to add more modules here?
     "--output", "/jlinked"]           #   Use jdeps to find out.


FROM alpine

# This is the line from AdoptOpenJDK:
RUN apk --update add --no-cache ca-certificates curl openssl binutils xz \
    && GLIBC_VER="2.28-r0" \
    && ALPINE_GLIBC_REPO="https://github.com/sgerrand/alpine-pkg-glibc/releases/download" \
    && GCC_LIBS_URL="https://archive.archlinux.org/packages/g/gcc-libs/gcc-libs-8.2.1%2B20180831-1-x86_64.pkg.tar.xz" \
    && GCC_LIBS_SHA256=e4b39fb1f5957c5aab5c2ce0c46e03d30426f3b94b9992b009d417ff2d56af4d \
    && ZLIB_URL="https://archive.archlinux.org/packages/z/zlib/zlib-1%3A1.2.9-1-x86_64.pkg.tar.xz" \
    && ZLIB_SHA256=bb0959c08c1735de27abf01440a6f8a17c5c51e61c3b4c707e988c906d3b7f67 \
    && curl -Ls https://alpine-pkgs.sgerrand.com/sgerrand.rsa.pub -o /etc/apk/keys/sgerrand.rsa.pub \
    && curl -Ls ${ALPINE_GLIBC_REPO}/${GLIBC_VER}/glibc-${GLIBC_VER}.apk > /tmp/${GLIBC_VER}.apk \
    && apk add /tmp/${GLIBC_VER}.apk \
    && curl -Ls ${GCC_LIBS_URL} -o /tmp/gcc-libs.tar.xz \
    && echo "${GCC_LIBS_SHA256}  /tmp/gcc-libs.tar.xz" | sha256sum -c - \
    && mkdir /tmp/gcc \
    && tar -xf /tmp/gcc-libs.tar.xz -C /tmp/gcc \
    && mv /tmp/gcc/usr/lib/libgcc* /tmp/gcc/usr/lib/libstdc++* /usr/glibc-compat/lib \
    && strip /usr/glibc-compat/lib/libgcc_s.so.* /usr/glibc-compat/lib/libstdc++.so* \
    && curl -Ls ${ZLIB_URL} -o /tmp/libz.tar.xz \
    && echo "${ZLIB_SHA256}  /tmp/libz.tar.xz" | sha256sum -c - \
    && mkdir /tmp/libz \
    && tar -xf /tmp/libz.tar.xz -C /tmp/libz \
    && mv /tmp/libz/usr/lib/libz.so* /usr/glibc-compat/lib \
    && apk del binutils \
    && rm -rf /tmp/${GLIBC_VER}.apk /tmp/gcc /tmp/gcc-libs.tar.xz /tmp/libz /tmp/libz.tar.xz /var/cache/apk/*

COPY --from=jlink /jlinked /opt/jdk/

## Add your application here, and change the CMD below to start it

CMD ["/opt/jdk/bin/java", "-version"]
```

Personally, I rather hope that Portola can graduate to GA, as we'll have an easier way to create images, not to mention 48mb less in them. If you are interested, please [download it](https://jdk.java.net/12) and try it out - report any bugs you find, and if you don't find any then Yay! Tell your friends!

\[Thank you to Portola Project Lead, [Mikael Vidstedt](https://twitter.com/mikaelvidstedt) for proof-reading this post\]
