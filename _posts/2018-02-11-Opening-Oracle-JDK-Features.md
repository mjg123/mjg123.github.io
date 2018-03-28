---
layout:     post
title:      Opening the Oracle JDK
date:       2018-02-11 10:00:00
summary:    Roundup of Oracle JDK features being added to OpenJDK
tags:
- draft
- jvm
---

At Java One in Oct 2017 Mark Cavage announced that [Oracle will be open-sourcing all the proprietary features of the Oracle JDK](https://youtu.be/Tf5rlIS6tkg?t=567) so that there will be "zero differences" between Oracle and Open JDK. This got a big round of applause at the time, and another when Mark Reinhold repeated that pledge in his [State of OpenJDK](https://fosdem.org/2018/schedule/event/state_openjdk/) talk at FOSDEM Feb 2018. Those who are paying customers of Oracle might be familiar with some of the features in question, as will the developers who work on the JDK itself. But I think the vast majority of Java programmers worldwide will not be so knowledgeable. Until recently I had to include myself in that group but after the announcements above I got curious.

I have been lucky enough to be able to speak to several of the people involved in the work of "Opening the Oracle JDK". I hope I can explain some of the less well-known things, and point out some cool new stuff which will be added to OpenJDK.

Internally the Java teams are tracking around ?HOW MANY? (mjg: check this) specific items. Some are trivial and some are destinied for deprecation rather than contribution. However there are several which I think will be really important contributions to OpenJDK, and most Java users should know about them.

So what are they? Well, for clarity lets split up the JDK into 4 parts: **Java the Language**, the **Core Java Libraries**, **Tooling**, and **The JVM** itself.

### Java the Language
Java is the same everywhere. There are no Oracle-only language features of Java.

### Core Java Libraries
`java.util.Hashmap` and so on are exactly the same on Oracle and Open JDKs. There are some libraries distributed in the `com.oracle` package with Oracle JDK, but these will be either deprecated or moved to standalone libraries. I didn't find anything eye-catching for the average Java user amongst these.

### Tooling
JDK tooling includes several end-user things like `jlink`, as well as the tools needed to build and distribute the JDK itself. Packaging the JDK to create installers for MacOS and Windows uses some software whose licenses make it hard to open-source so there will be some changes there (notice that you can download `exe` and `dmg` installers for Oracle JDK from [http://jdk.java.net/10/](http://jdk.java.net/10/) but OpenJDK is all `.tar.gz`). This is very important but not something that Java programmers think about very often. The the timezone updater has some interesting history, and JMC is well worth knowing about. There are also some changes to testing frameworks and how secutiry vulnerabilities are handled.

### The JVM
This is where the vast majority of the Oracle-JDK differences are. Oracle is open-sourcing a new GC, performance enhancements, monitoring tools and a host of smaller changes.



## The features in detail

So here is my roundup of interesting features. Remember that the new 6-monthly release cycle means that things are released when they are ready so there are no commitments about which release a feature will be ready for. It is hoped to not have anything large outstanding after JDK12.

### Garbage Collection

#### ZGC

[ZGC](https://wiki.openjdk.java.net/display/zgc/Main) is a new Garbage Collector designed to handle very large heap sizes (several terabytes) with predictable pause lengths (single-digit miliseconds). Also Z is intended to be a platform for future work on large-heap GCs.

[Stefan Karlsson](https://twitter.com/stekarmatrik) and [Per Liden](https://twitter.com/perliden) have been working on ZGC since its inception and presented a deep dive at JFokus this year:

<iframe width="700" height="400" src="https://www.youtube.com/embed/tShc0dyFtgw" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

#### G1GC improvements

"Garbage First" GC is _not_ a new GC. It has been the default GC since Java 9. One of the goals of G1 is to respect a user-supplied maximum pause time, and it uses some heuristic to calculate how much work to do during each pause to meet this goal. This heuristic _may_ overestimate how much work to do, in which case pauses can overrun. This work is to make such pauses "abortable", and the end result should be that G1 is more reliable for low-latency use. This feature is known as [Abortable Mixed Collections](https://bugs.openjdk.java.net/browse/JDK-8190269).

G1 is still the JDKs default general-purpose GC so there are also many other startup and overhead improvements to G1 in progress as well.


### AppCDS

Application Class Data Sharing is one of the most exciting items, for me. AppCDS can dramatically improve application startup and memory usage. I wrote about Application CDS before [here]({% post_url 2017-10-04-AppCDS-and-Clojure %}). It was contributed completely in Nov 2017 and will be available in OpenJDK 10.

### TZ updater

[Dealing with times and dates](http://infiniteundo.com/post/25326999628/falsehoods-programmers-believe-about-time) is a [constant source of confusion](https://stackoverflow.com/questions/6841333/why-is-subtracting-these-two-times-in-1927-giving-a-strange-result) for developers. Why can't [humans](https://www.timeanddate.com/worldclock/new-zealand/chatham-islands) (and [astronomical bodies](http://tycho.usno.navy.mil/leapsec.html)) just be consistent?! Until that happens we will have to deal with it, and one of the fun things about it is how political and social forces can push things around. Timezones and DST can be changed with surprisingly little warning, so a runtime which provides date/time support can't possibly know in advance how to behave in all cases. A couple of fun examples:

  - Australian 2000 Olympics and 2006 Commonwealth Games both moved DST clock-change dates by about a week to "avoid confusion", with [only a few months notice](https://scott.yang.id.au/2006/03/commonwealth-games-dst.html).
  - Israel's Knesset used to [decide the dates of DST at the last moment](http://self.gutenberg.org/articles/eng/Israel_Summer_Time). There was even a suggestion in 2010 to move to winter-time for a single day during DST (which didn't happen in the end). This kind of confusion actually led to Microsoft temporarily abandonning Israel Daylight Time for Windows and making everyone's Outlook an hour off. These days IDT dates are fixed, thankfully.

Oracle JDK customers have had a tool called [TZ Updater](http://www.oracle.com/technetwork/java/javase/tzupdater-readme-136440.html) to keep their installation up to date. OpenJDK users have been able to use [an open tool provided by Azul](https://www.azul.com/products/open-source-tools/ziupdater-time-zone-tool/) for some time.


### Font-rendering engine

In order to quickly and accuratly render font glyphs, Oracle and Open JDKs both lean on services provided by the OS (eg [CoreText](https://developer.apple.com/documentation/coretext) on MacOS). Additionally OracleJDK bundles a version of the T2K rendering engine which is proprietary and was licensed by Sun. OpenJDK is not able to redistribute T2K and instead bundles [FreeType](https://www.freetype.org/). I found [some historical info](http://openjdk.java.net/projects/font-scaler/), and [here's where the code diverges](http://hg.openjdk.java.net/jdk10/hs/file/d85284ccd1bd/src/java.desktop/share/classes/sun/font/FontScaler.java#l99).

Here's an image showing OracleJDK (top) vs OpenJDK (bottom) rendering some text:

![different?]({{ "assets/JDK-font-rendering.png" | absolute_url }})

There is no difference. In fact the JDK mainly uses the OS font services, but in the case of TrueType fonts loaded from a stream at runtime T2K or FreeType might be used. If this change causes some of your tests to fail after OracleJDK switches to FreeType (a change is already in the JDK11 codebase) then you have my sincere sympathy.

### Java Flight Recorder and Java Mission Control

Java Flight Recorder (JFR) is a profiling tool. Java Mission Control (JMC) is for viewing the output of JFR. I expect these to become standard tools for profiling JVM workloads and diagnosing performance problems.

JFR works as a high-performance event recorder built into the JVM, lightweight enough to be left always-on (goal: no more than ~1% overhead). It's been bundled with OracleJDK since 2013 and was in development by JRockit for some time before that, so it is mature and reliable and has been battle-tested by several of Oracle's customers.

JFR is very flexible with rules and triggers about what to collect and when, eg a CPU% threshold is reached. Because it is *part* of the JVM, it doesn't need to use the [JVM TI](https://en.wikipedia.org/wiki/Java_Virtual_Machine_Tools_Interface) which saves some overhead and helps avoid problems like safe-point bias. I don't speak with much authority about JVM internals and profiling but I have found [Nitsan Wakart's Psy-Lob-Saw blog](http://psy-lob-saw.blogspot.co.uk/2015/12/safepoints.html) very instructive.

In development it is permitted to use OracleJDK commercial features, so you can try out JFR today. It's behind a couple of args: `-XX:+UnlockCommercialFeatures -XX:+FlightRecorder`. This [getting started guide](https://medium.com/@chrishantha/using-java-flight-recorder-2367c01deacf) is a little dated but helpful. In the end I hope JFR will be added to JDK 11 or 12 but there are no commitments yet.

Here is [a proposal to open Flight Recorder](http://openjdk.java.net/jeps/328), and [a proposal for a proposal for a new OpenJDK project for Mission Control](http://mail.openjdk.java.net/pipermail/discuss/2018-March/004715.html). Note that "Java" has been dropped from both names.

Here's JFR/JMC lead [Marcus Hirt](https://twitter.com/hirt) talking about them at JFokus a few years ago:

<iframe width="700" height="400" src="https://www.youtube.com/embed/EOeNugl0NlU" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

### Tonga Tests

I admit this won't affect your life as a Java programmer, but I thought it was an interesting situation which has come about through quite normal bureacracy.

Tonga is a testing framework which is used at Oracle to run some tests of JDK functionality, both Oracle-proprietary and open. As is common in testing code, there is a dependency from the test code itself on the framework. Unfortunately Tonga contains some licensed (non-Oracle) code which Oracle is not able to distribute. So, in fact _Oracle maintains some closed tests of open functionality_. Of course this looks rather inefficient, and indeed Oracle engineers are rewriting these tests and contributing them to OpenJDK (TODO: Any evidence of this?!). The result should be more reliable code and less overhead for JVM engineers.

Oracle also maintains some vulnerabilty tests internally which will hopefully be moved to the [OpenJDK Vulnerability Group](http://cr.openjdk.java.net/~mr/ojvg/) (once it exists).

## Summary

As I said at the start, it seems like a popular move for Oracle to contribute to open-source. And I think the JDK features I've written about here will be welcome additions that people can use for free, as well as it being good to remove some confusion about what is the difference between Oracle and Open JDKs. There will still be an OracleJDK, for the purposes of offering commercial support, but it will be the exactly same functionally as OpenJDK.
The other, even larger change to Java which was announced at the same time was the change to a 6-monthly release cycle. As features are not committed to a release until they are ready, they must be held in a branch which increases the testing burden on engineers and their testing infrastructure. So I think that stopping carrying proprietary extensions to OpenJDK will be helpful for the smoothness of the new release cycle too.

Every JDK engineer I spoke to is proud of their work, and everyone is happy to be able to have it more widely used. I hope that the developers among you who target the JVM get a chance to try these new features as they arrive.
