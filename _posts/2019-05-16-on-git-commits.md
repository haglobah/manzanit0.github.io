---
layout: post
title: "On git commits"
author: Javier García
description: "What happens when you write clean git commits"
category: git
apprenticeship: false
tags: git, clean
---

Just yesterday I was reading Tim Pope's
[note about git commit messages](https://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html),
and I really liked it. What I wanted to share though was my surprise today, when I signed in Github to
check my git history after having tried to follow his template during the morning.

### In a nutshell

Summarizing Tim's note as much as possible (although I encourage you to read his post better),
the suggestions basically are:

1. Start your commit message with a **summary phrase, no more than 50 characters**.
2. Add any **explanations below**, after a couple of line breaks
3. **Wrap the message** yourself to around 72 characters, *'cause git ain't gonna do it for ya*.

The cool thing about this is that this isn't just about having `git log` display the messages in a nice format,
which it does, but that further integrations with git also work better, Github for example.

These are two examples from my work this morning. In the first one, you can see how, by making
the summary way too long, you can barely understand it, since it breaks.

![Bad message](/assets/on-git-commits/bad-git-commit-message.png)

On the other hand, these nicely written commits have their summary properly displayed and the description
when you click on the three dots.

![Good message](/assets/on-git-commits/nice-git-commit-message.png)

As you can see the difference is huge, so even if it's just to be able to browse our git history
from Github and be able to read all the explanations nicely, it's worth the effort.
