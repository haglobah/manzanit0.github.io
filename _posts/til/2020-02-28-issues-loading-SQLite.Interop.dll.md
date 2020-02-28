---
layout: post
title: "Issues loading SQLite.Interop.dll"
author: Javier Garcia
category: SQLite
tags: c#, .NET, sqlite
---

Recently, while developing a small CLI application for Windows, I decided to
add SQLite for some caching. To my surprise, after finishing the coding, when
I decided to start adding some tests, I came across the following error:

> Unhandled Exception: System.DllNotFoundException: Unable to load DLL
> 'SQLite.Interop.dll'

I'll cut the whole story short: **You need to add the SQLite assembly to the
entry project that uses it, even if the project doesn't actually use any of
the packages' classes**. As soon as you add the nugget dependency, it stops
crying. I found this a tricky problem because the compiler doesn't complain,
since you don't use any classes - it's just a runtime issue.
