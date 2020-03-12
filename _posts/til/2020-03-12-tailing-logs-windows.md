---
layout: post
title: Tailing logs in Windows
author: Javier Garcia
category: Logging
tags: Logging, Windows
---

In a Unix environment, usually the preferred way to _tail_ logs is to use the `tail` command. However
in Windows, it's slightly different. Today I just wanted to share two alternatives I've found
when looking for a way to do this.

## Powershell

In case you want to use already existing tools in Windows, PowerShell is a really good option.
PowerShell 3 includes a `-Tail` parameter to include the amount of lines you need.

```ps1
Get-Content filenamehere -Wait -Tail 30
```

## BareTail

If you're more of a GUI kind of person, then [BareTail][0] might be what you're looking for.
It provides with some basic highlighting capabilities and it's extremely easy to use.

[0]: http://www.baremetalsoft.com/baretail/