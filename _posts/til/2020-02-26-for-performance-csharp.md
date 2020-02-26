---
layout: post
title: "for vs foreach performance in .NET"
author: Javier Garcia
category: c#
tags: c#, .NET, performance
---

Today I Learnt the actual performance differences between a `for` and a `foreach` loop in .NET.
While it was evident for me that the `for` loop would be more performant in most scenarios, I
always thought that maybe the compiler had some form of optimization to not use the iterators when
possible. Looks like not.

**TLDR:**

> Using `for` loops on List are a bit more than 2 times cheaper than `foreach` loops on List.
>
> Looping on array is around 2 times cheaper than looping on List.
>
> As a consequence, looping on array using for is 5 times cheaper than looping on List using `foreach`.

**Long version by Patrick Smacchia:**

[An easy and efficient way to improve .NET code performances][0]

[0]: http://codebetter.com/patricksmacchia/2008/11/19/an-easy-and-efficient-way-to-improve-net-code-performances/
