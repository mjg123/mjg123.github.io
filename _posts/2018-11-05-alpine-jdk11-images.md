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

If you are a conscientious container-image-builder, you will have heard many times the advice to keep images small. From Google (link), from me (link), from Red Hat or from someone else.  The reasons are numerous: small images are faster to copy around, faster to start, and by removing unneeded things you are reducing the damage that a security exploit could do. If you feel you need debugging tools in your container, you should be using sidecars (ie, container images containing debug tools that can be attached to running containers). Everything in your container images should earn its space.

## Alpine, Musl and the JVM

For many use-cases, Alpine Linux (link) is the ideal base image to use. First of all it's very small - 4.2mb (todo: check), which is about 3% of a typical "slim" version of a regular Linux distro. However, there is something you have to know about Alpine: it does not use GLibC (todo: check capitalization), instead they use Musl (link). What are glibc and musl? They are programming APIs which provide user code with access to Linux Kernel functionality, such as opening files or network connections. Although they are abstractions over the same underlying thing (the Linux Kernel binary interface), they expose slightly different APIs to a programmer. So C or C++ code which is compiled against glibc will not run on a musl system, and vice-versa. In fact, the code will need to be (slightly) different depending on which you are targetting.

Why am I telling you, dear Java programmer, about C and C++ Kernel APIs?  Well, because you use an application written in C++ to do a lot of your work: The JVM!  And how the JVM interacts with the Linux kernel is critical to how it can be used in containers.  If you want to put your Java or Clojure or Kotlin or Scala application in an Alpine Linux container, you need to be able to run your JVM on musl!

## Project Portola

As mentioned above, there are some code changes needed in the JVM to run on musl-based systems - and such changes do exist in the repository of [Project Portola](https://openjdk.java.net/projects/portola/).  Before JDK 11 was released it was possible to get early-access builds of the Portola codebase, but it did not graduate to a "General Availability" release, and now you cannot download Portola builds of JDK 11. There are early-access builds of JDK 12 for Alpine, based on Portola, at the [JDK12-ea downloads page](http://jdk.java.net/12/).  I have not found any builds of OpenJDK 11 for Alpine publicly available from any vendor.

But, don't worry, there is a workaround.

## Glibc Compatibility for JVM on Alpine

[Sasha Gerrand](https://github.com/sgerrand) maintains a glibc package for Alpine Linux. If you add this package to an Alpine system, you will be able to run Glibc-based applications on Alpine.  This is how the [AdoptOpenJDK Alpine images](https://github.com/AdoptOpenJDK/openjdk-docker#supported-builds-and-build-types) are produced - they do not use a musl port of the JVM.  For the most part you can just take one of the AdoptOpenJDK images and run your application as usual, enjoying the 100+ megabyte image size reduction.  In fact the JVM requires a few more packages to be added to a base Alpine image. The total size of Alpine plus all the required libs, plus the glibc-compatibility layer is 52mb. This is a lot bigger than the base Alpine image, but still a lot smaller than ubuntu-slim, for example.

## Using jlink to shrink the JDK

If you use the AdoptOpenJDK Alpine images to deploy your JVM-based applications then the largest component in the images is likely to be the JDK itself. And you are _highly unlikely_ to be using all of the features that the JDK provides.  I've written about using `jlink` to shrink JDK distributions [before](todo:link) - but the TL;DR is that `jlink` can provide a JDK distribution which contains _only_ the Java modules you need. If you are using Portola-based JDKs then the smaller JDK which `jlink` creates will be fine to run on Alpine as-is, and you can have a Dockerfile like this:

```
FROM alpine:latest as build
ADD openjdk-11+28_linux-x64-musl_bin.tar.gz /opt/jdk  # This is the Portola JDK distribution
RUN ["/opt/jdk/jdk-11/jlink", "--compress=2", \
     "--module-path", "/opt/jdk/jdk-11/jmods", \
     "--add-modules", "java.base", \
     "--output", "/jlinked"]


FROM alpine:latest
COPY --from=build /jlinked /opt/jdk/
ADD HelloWorld.class /
CMD ["/opt/jdk/java", "HelloWorld"]
```

However, if you try using `adoptopenjdk/openjdk-11:alpine-slim` as your base image for the first stage, you won't need to use the `ADD` line - but the resulting image _won't work_.  It will fail with a rather confusing `not found` error. The error is because the `alipine-slim` JDK builds use the regular glibc-based JVMs so copying them into a plain `alpine:latest` image won't work.  You need to _manually_ install the glibc-compatibility package in your final image, as well as the extra libs that the JVM needs (libssl, zlib and a few others).

The easiest way to do that right now is to copy how AdoptOpenJDK have done it, which you can find in this rather beefy `RUN` command in their [Alpine Dockerfiles](https://github.com/AdoptOpenJDK/openjdk-docker/blob/2baf4481c1a3a70f47a8aae074ec9a4027945638/11/jdk/alpine/Dockerfile.hotspot.releases.slim#L24-L46).  A Dockerfile which you can build from is as follows:

```Dockerfile
FROM adoptopenjdk/openjdk11:alpine-slim AS jlink
RUN ["jlink", "--compress=2", \
     "--module-path", "/opt/java/openjdk/jmods", \
     "--add-modules", "java.base", \   # Maybe you need to add more modules here?
     "--output", "/jlinked"]           #   Use jdeps to find out.


FROM alpine

# This is the line that AdoptOpenJDK use:
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
