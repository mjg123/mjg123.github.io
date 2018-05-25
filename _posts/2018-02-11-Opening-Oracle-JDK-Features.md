---
layout:     post
title:      Opening the Oracle JDK
date:       2018-02-11 10:00:00
summary:    Roundup of Oracle JDK features being added to OpenJDK
tags:
- draft
- jvm
---

At Java One in Oct 2017 Mark Cavage announced that [Oracle will be open-sourcing the proprietary features of the Oracle JDK](https://youtu.be/Tf5rlIS6tkg?t=567). This got a big round of applause at the time, and Mark Reinhold got another for repeating that pledge in his [State of OpenJDK](https://fosdem.org/2018/schedule/event/state_openjdk/) talk at FOSDEM Feb 2018. Donald Smith, Sr Director of Product Management in the Java Platform Group writes ["our intent is that within a few releases there should be no technical differences between OpenJDK builds and Oracle JDK binaries"](https://blogs.oracle.com/java-platform-group/faster-and-easier-use-and-redistribution-of-java-se).

Some of you may be familiar with some of the features in question, but I think the vast majority of Java developers are not. I got curious and decided to do some research, and this post is a summary of major features being open-sourced.

For clarity we can split up the JDK into 4 parts: **Java the Language**, the **Core Java Libraries**, **Tooling**, and **The JVM** itself.

### Java the Language
Java is the same everywhere. There are no Oracle-only language features of Java.

### Core Java Libraries
Core libraries like `java.util.Hashmap` and so on are exactly the same on Oracle and Open JDKs. There are some libraries distributed in the `com.oracle` package with Oracle JDK, but these will be either deprecated or moved to standalone libraries. I didn't find anything eye-catching for the average Java user amongst these, so we can consider the effect to be no change to the core libraries.

### Tooling
JDK tooling includes tools like [`jlink`](https://mjg123.github.io/2017/11/07/Java-modules-and-jlink.html), as well as the tools needed to build and distribute the JDK itself.

The timezone updater has some interesting history, and Mission Control is well worth knowing about. There are also some changes to testing, and to the installers (notice that you can download `exe` and `dmg` installers for Oracle JDK but [OpenJDK](http://jdk.java.net/10/) is all `.tar.gz`).


### The JVM
This is where the majority of the OracleJDK-only features are. Oracle is open-sourcing a new GC, performance enhancements, tracing tools and a host of other changes.


## The features in detail

So here is my roundup of interesting features.

### ZGC

[ZGC](https://wiki.openjdk.java.net/display/zgc/Main) is a new Garbage Collector designed to handle very large heap sizes (several terabytes) with predictable pause lengths (single-digit miliseconds). Also Z is intended to be a platform for future work on large-heap GCs.

[Stefan Karlsson](https://twitter.com/stekarmatrik) and [Per Liden](https://twitter.com/perliden) have been working on ZGC since its inception and presented a deep dive at JFokus this year:

<iframe width="700" height="400" src="https://www.youtube.com/embed/tShc0dyFtgw" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

ZGC early access builds are [now available](http://jdk.java.net/zgc/).

### AppCDS

Application Class Data Sharing is one of the most exciting items, for me. AppCDS can dramatically improve application startup and memory usage. I wrote about Application CDS before [here]({% post_url 2017-10-04-AppCDS-and-Clojure %}). It was open-sourced completely in Nov 2017 and was released in OpenJDK 10.

### TZ updater, Usage Logger

[Dealing with times and dates](http://infiniteundo.com/post/25326999628/falsehoods-programmers-believe-about-time) is a [constant source of confusion](https://stackoverflow.com/questions/6841333/why-is-subtracting-these-two-times-in-1927-giving-a-strange-result) for developers. Why can't [humans](https://www.timeanddate.com/worldclock/new-zealand/chatham-islands) (and [astronomical bodies](http://tycho.usno.navy.mil/leapsec.html)) just be consistent?! Timezones and DST dates are changed with surprisingly little warning, so a runtime which provides date/time support can't possibly know in advance how to behave in all cases. A couple of fun examples:

  - Australian 2000 Olympics and 2006 Commonwealth Games both moved DST clock-change dates by about a week to "avoid confusion", with [only a few months notice](https://scott.yang.id.au/2006/03/commonwealth-games-dst.html).
  - Israel's Knesset used to [decide the dates of DST at the last moment](http://self.gutenberg.org/articles/eng/Israel_Summer_Time). There was even a suggestion in 2010 to move to winter-time for a single day during DST (which didn't happen in the end). These days IDT dates are fixed, thankfully.
  - North Korea's timezone recently changed with [4 day's notice](https://en.wikipedia.org/wiki/Time_in_North_Korea#History).

Oracle JDK customers have had a tool called [TZ Updater](http://www.oracle.com/technetwork/java/javase/tzupdater-readme-136440.html) to keep their installation up to date with the latest rules. TZ Updater is part of the infrastructure which is planned to be open-sourced, along with other tools such as the Java Usage Logger which allows companies to gather data on how they use the JVM.


### Font-rendering engine

In order to quickly and accuratly render font glyphs, Oracle and Open JDKs both lean on services provided by the OS (eg [CoreText](https://developer.apple.com/documentation/coretext) on MacOS). Additionally OracleJDK bundles a version of the T2K rendering engine which is proprietary. OpenJDK does not redistribute T2K and instead bundles [FreeType](https://www.freetype.org/). I found [some historical info](http://openjdk.java.net/projects/font-scaler/), and [here's where the code diverges](http://hg.openjdk.java.net/jdk10/hs/file/d85284ccd1bd/src/java.desktop/share/classes/sun/font/FontScaler.java#l99).

Here's an image showing OracleJDK (top) vs OpenJDK (bottom) rendering some text:

![different?]({{ "assets/JDK-font-rendering.png" | absolute_url }})

There is no difference. In fact both JDKs use OS font services most of the time, but in the case of TrueType fonts loaded from a stream at runtime T2K or FreeType might be used. If some of your tests fail after OracleJDK switches to FreeType (this change is already in the JDK11 codebase) then you have my sincere sympathy.

### Flight Recorder and JDK Mission Control

[Flight Recorder](https://docs.oracle.com/javacomponents/jmc-5-4/jfr-runtime-guide/about.htm#JFRUH170) (FR) is a profiling tool. [JDK Mission Control](http://www.oracle.com/technetwork/java/javaseproducts/mission-control/java-mission-control-1998576.html) is for viewing the output of FR. These tools are [regarded as being very powerful](https://news.ycombinator.com/item?id=16599567), and I personally expect them to become standard tools for profiling JVM workloads and diagnosing performance problems. NB there is some change to the name of these features - as proprietary features they both had "Java" in the name.

FR works as a high-performance event recorder built into the JVM, lightweight enough to be left always-on (goal: no more than ~1% overhead). It's been bundled with OracleJDK since 2013 and was in development by JRockit for some time before that, so it is mature and reliable and has been battle-tested by several of Oracle's customers.

FR is very flexible with rules and triggers about what to collect and when. For example we could set a rule to start recording when a CPU% threshold is reached. And, because it is *part* of the JVM, it doesn't need to use the [JVM TI](https://en.wikipedia.org/wiki/Java_Virtual_Machine_Tools_Interface) which saves some overhead and helps avoid problems like safe-point bias. I don't speak with as much authority about JVM internals and profiling as I would like, but I have found [Nitsan Wakart's Psy-Lob-Saw blog](http://psy-lob-saw.blogspot.co.uk/2015/12/safepoints.html) very instructive.

In development it is permitted to use OracleJDK commercial features, so you can try out FR today. It's behind a couple of args: `-XX:+UnlockCommercialFeatures -XX:+FlightRecorder`. This [getting started guide](https://medium.com/@chrishantha/using-java-flight-recorder-2367c01deacf) is a little dated but helpful.

Here is the JEP describing [Flight Recorder](http://openjdk.java.net/jeps/328) which will be included in JDK11, and the new OpenJDK project: [Mission Control](http://openjdk.java.net/projects/jmc/).

Project lead [Marcus Hirt](https://twitter.com/hirt) spoke about them at JFokus a few years ago:

<iframe width="700" height="400" src="https://www.youtube.com/embed/EOeNugl0NlU" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

### CA Certificates

OpenJDK included a set of root Certificate Authority certificates for the first time in the JDK10 release.  Prior to 10, OpenJDK did not contain any of these certs so it was impossible to establish trusted and secure relationships using (eg) HTTPS without finding and importing certificates yourself. This change is a great help to developers who wish to develop secure applications on OpenJDK, and is discussed in more depth on [this post by James Connors](https://blogs.oracle.com/jtc/openjdk-10-now-includes-root-ca-certificates).

### JVM Tests

Tonga is a testing framework which is used by Oracle to run some tests of JDK functionality, both Oracle-proprietary and open. As is common in testing code, there is a dependency from the test code to the framework. However, Tonga contains some code which Oracle can't release under an open-source license. So, in fact Oracle engineers have been maintaining some closed tests of open functionality. Engineers have been porting and writing new tests in OpenJDK - here's a series of announcements which in total add up to 900,000 lines of test code being added to OpenJDK: 
[1](https://twitter.com/OpenJDK/status/996000521319313409)
[2](https://twitter.com/OpenJDK/status/996000159870914560)
[3](https://twitter.com/OpenJDK/status/995999825781981184)
[4](https://twitter.com/OpenJDK/status/995999644747489280)
[5](https://twitter.com/OpenJDK/status/995999471753420800)
[6](https://twitter.com/OpenJDK/status/995999267293712384)
[7](https://twitter.com/OpenJDK/status/995999056399872000)

## Summary

There will still be an OracleJDK for the purposes of offering commercial support, but it will be functionally the same as OpenJDK.

I am personally delighted to see Oracle contribute even more open-source - I work for Oracle on [an open-source project](https://fnproject.io) - and I hope you agree that the features I've described here are useful additions to OpenJDK. There are alse a couple of useful side-effects: firstly it removes confusion about the differences between OracleJDK and Open JDK, and secondly it reduces the burden on JDK developers who have to maintain feature branches. Because of the new six-monthly release cycle features cannot be committed early to a release, they must be held in a branch until ready. Having fewer combinations of things to test can only be beneficial to the agility of the JDK.

Every JDK engineer I spoke to is proud of their work, and everyone is happy to be able to have it more widely used. I hope that the developers among you who use the JVM get a chance to try these new features as they arrive.


## Credits

I would like to extend special thanks to [Dalibor Topic](https://twitter.com/robilad) for helping me prepare this post. I am also extremely grateful to the numerous JDK engineers and other members of the Java Platform Group who I have spoken to over the last few months.
