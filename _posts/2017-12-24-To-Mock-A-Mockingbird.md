---
layout:     post
title:      To Mock a Mockingbird
date:       2017-12-24 10:00:00
summary:    To Mock A Mockingbird
tags:
- maths
- not-computers
---

![Smullyan]({{ "assets/smullyan2.jpg" | absolute_url }})


> Why should I worry about dying? It's not going to happen in my lifetime! - Raymond Smullyan

This quote by [Raymond Smullyan](https://en.wikipedia.org/wiki/Raymond_Smullyan) shows very nicely his wit and playful love of logic. Smullyan was a concert pianist, logician, close-up magician, philosopher and a prolific author of mathematics, both academic and recreational. He died, not in his own lifetime of course, in February 2017 aged 97.


## Puzzles and Books

Smullyan wrote a lot of puzzle books which introduce a curious reader to more and more devilishly difficult logic puzzles. Usually they involve Knights and Knaves (you know the kind: they always tell the truth or always lie, but are otherwise indistiguishable) Vampires and other creatures with weird *but logically consistent* behaviour. In this way he is forcing you the reader to probe the dustier corners of what can and can't be known from given premises.

A lot of these problems can be solved by carefully applying if-this-then-that inference, truth-tables and patience. They are fun, to be sure, and a great mental workout. However, I would like to write a little about one book which is slightly different from those. A book which I am currently working through and enjoying immensely.

## To Mock a Mockingbird

To Mock a Mockingbird is two books. Or rather, a book in two halves. The first half is logic puzzle set on islands of knights and knaves, fussy barbers, Russell's paradox, unrealistic prenuptials and so on. Great fun, but the second half of the book - which features the mockingbird - is where it really takes off.

Mockingbird is Smullyan's retelling of *combinator theory*. I will say right now that I am not any kind of expert in combinatory theory, and I will add that that fact hasn't stopped me enjoying this book a good deal.

The premise is that there are **enchanted forests** which contain many (or sometimes very few) **talking birds**. Smullyan dedicated the book to [Haskell Curry](https://en.wikipedia.org/wiki/Haskell_Curry) - *an early pioneer in combinatory logic and an avid bird-watcher*. The birds, which I suppose represent the combinators, have an interesting characteristic:

> Given any two birds A and B, if you call out the name of B to A it will respond by calling out the name of some bird to you.

This bird whose name `A` calls when you call `B` is denoted as `AB`. Once you have several birds in place, a single call can cascade around the forest with each call following rules depending on who produces it.

The very first bird we are introduced to is the *Mockingbird* whose characteristic behaviour is that whatever name you call to the Mockingbird, it will reply as if it is the bird whose name you called. This is denoted:

`Mx = xx`

For *any* bird `x` we can say that `Mx` (the result of calling `x` to a Mockingbird) is the same as `xx` (the result of calling `x` to a bird of type `x`). It really does mock other birds!

## Birds birds birds

Soon we discover that birds have certain properties: The can be **fond of** other birds, they can be **egocentric** if they are fond of themselves. The can be **hopelessly egocentric** if they only ever talk about themselves. There are **happy** birds, **normal** birds, **agreeable** birds and many others. We also meet other types of birds with specific properties - the **Lark**, the **Kestrel**, **Sage** birds, **Bluebirds**, **aristocratic** birds, **Eagles**, the list goes on and on. Luckily there is a Who's Who list of birds in the back to keep track.

All of this is really delightful - Smullyan has a real knack for characterising these things in extraordinary and clever ways. For example a typical question might read:

> Why is a hopelessly egocentric Lark unusually attractive?

(SPOLIER: the rules of the forest imply that *all* birds must be **fond of** a hopelessly egocentric Lark - but the fun is in the proof so I haven't spoilt it for you really)

And it's not just birds for their own sake. *Sage* birds are fixed-point combinators, for example. Later chapters take us to Curry's forest and onwards where we explore Church-Turing completeness and Gödel's incompleteness theorems. I'm looking forward to it.

## Happy Holidays 2017

As you can tell, I'm enjoying this book a lot. Hopefully I'll get to understand every bird in the forest by the time my xmas break is over. Thanks again Mr Smullyan.
