---
layout:     post
title:      Comparing Conference Proposals
date:       2018-11-06 00:00:00
summary:    One Accept, One Reject
tags:
- conference
- draft
---

This year I have submitted proposals to many conferences. My accept rate is around two-thirds, so I am quite used to having talks rejected. I have never received any feedback about rejections - and I don't think I can expect to, as conference organisers have enough work to do without spending the time to provide that kind of in-depth feedback. I am always deeply grateful for ever getting a chance to speak and cannot bear any ill-will to a conference for rejecting me.

So, wondering _why_ our talks have been rejected is generally left up to us, the proposers. One interesting experiment might be possible to arrange if we submitted multiple talks to a conference and had a mix of acceptance and rejection, and in fact that happened to me at [JavaZone](https://2018.javazone.no/) - I submitted 2 and had one accepted and one rejected. Here I will put the two proposals, and below them I will discuss the reasons why I think they got the results they did:

## Two talks

> **Java in a World of Containers**
> 
> Container technologies such as Docker are becoming the most popular way to deploy cloud applications, and Java is committed to being a good container citizen. This talk will cover some of the new the tools and techniques for reducing container size, for improving startup time and sharing of data between containerized JVMs for better memory usage and speed. We'll see what was added in JDK 9 and 10 and have a look forward to what's coming in 11.

and

> **Java in Serverless Land**
>
> Serverless and FaaS are rapidly establishing themselves as fundamental cloud services. It's a radically simple model for deployment and management of code in the cloud, but while Java is regularly topping the usage charts for most types of server-side development uptake in Serverless is slower.
>
> In this talk I'll explore some of the reasons why that might be the case. Looking at the features of Java we'll see how the people who use it are (and aren't) served by Serveless platforms, and what we are doing to make serverless more Java-friendly in Fn Project, a new container-native open-source serverless compute platform developed publicly and sponsored by Oracle.

During this year I have given both of these talks at Java events, so I have a baseline level of confidence that they are talks that _can_ work.

## The Result

Which do you think would be accepted or rejected? In this case the first was accepted and the second wasn't.

## And Why?

I ran a "How to prepare for a conference proposal" workshop at work recently, using these as examples. The reasons we preferred the first over the second:

  - Containers are a more widely-used technology than Serverless, so there's a bigger audience.
  - The first talk describes concretely what the audience can expect to learn: "reducing container size, improving startup time, and sharing data between containerized JVMs" - wheras the second is much more vague: "explore some of the reasons why..."
  - Related to the above topic, the first seems more likely to have actionable things that people can actually use straight away.
  - By mentioning the project I work on and that it is sponsored by Oracle the reviewers minds might be edging towards "Is this a product pitch?"
  - "Serverless" was quite a popular buzzword at that time (and still is), so it might look like this is a low-effort talk designed to capitalize on that.
  - The second proposal seems a little scattered, on the one hand it's pretending to be a kind of "survey of the industry", and on the other it's completely about my own project. Unclear what they audience can expect to learn here.
  
## In summary

Every conference speaker can expect to get some rejections. I hope this pair of examples was useful to give you some thoughts about how to decrease the number that _you_ get.

By the way, I also learned later that JavaZone has a soft policy of not accepting two talks from the same speaker (as they are quite oversubscribed), so you have to also recognise that the reasons for a rejection might not be to do with the talk content at all...

