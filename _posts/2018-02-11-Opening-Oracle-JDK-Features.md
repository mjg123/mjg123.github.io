---
layout:     post
title:      Opening the Oracle JDK
date:       2018-02-11 10:00:00
summary:    Roundup of Oracle JDK features being added to OpenJDK
tags:
- draft
- jvm
---

At Java One in Oct 2017 Mark Cavage announced that Oracle will be open-sourcing all the proprietary features of the Oracle JDK so that there will be "zero differences" between Oracle and Open JDK. In fact that's only partly true. Several of the Oracle JDK features will be deprecated and removed. I have been lucky enough to be able to speak to several of the people involved in this work and would like to present what I have learned to you, dear reader.

Internally the Java teams are tracking around 50(?!) specific items, some trivial but several are amazing high-impact changes which will be really important contributions to OpenJDK.

So what are they? Well, for clarity lets split up the JDK into 4 parts: **Java the Language**, the **Core Java Libraries**, **Tooling**, and **The JVM** itself.

### Java the Language
Java is the same everywhere. There are no Oracle-JDK-only language features of Java.

### Core Java Libraries
`java.util.Hashmap` and so on are exactly the same on Oracle and Open JDKs. There are some libraries distributed in the `com.oracle` package with Oracle JDK, but these will be either deprecated or moved to standalone libraries. I didn't find anything eye-catching for the average Java user amongst these.

### Tooling
Java Web Start is already deprecated. Packaging the JDK to create installers for MacOS and Windows uses some software whose licenses make it hard to open-source so there will be some changes there. This is all very important (and legally tricky!) but not something that Java programmers think about very often. I will talk about the timezone updater below, and JMC.

### The JVM
This is where the vast majority of the Oracle-JDK differences are. Oracle is open-sourcing a new GC, performance enhancements, monitoring tools, testing frameworks and security processes .... (more here)



## The features, in detail

### ZGC, DetGC

### AppCDS

### TZ updater

### Font-rendering engine

(find example picture of difference. Commiserate anyone whose life is affected by this)

### JFR/JMC

### Tonga & Vulnerability tests

The OpenJDK vuln group exists now, find a link to it on the ML.


