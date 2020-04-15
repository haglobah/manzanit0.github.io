---
layout: post
title: Keybindings with iTerm
author: Javier Garcia
category: tooling
tags: tooling, iterm, terminal
---

Today, while trying to automate my compile/test/lint workflow in Go, I learnt
how easy it's to do [iTerm][0], with it's default settings.

Simply by going to `Settings > Keys > +` and adding a new binding, you just got
your new shortcut. In my case I associated `Ctrl + t` to:

```bash
go generate && go build -v && go test -v && go vet &&  golangci-lint run\n
```

Notice the `\n` at the end, so it runs, instead of simply _sending_ the text.

Here's a quick screenshot:

![screenshot](/assets/iterm-keybinding.png)

Now, while I can see how this is not very portable, for example to my Ubuntu machine,
It's so simple to set up, the ROI is nearly instantaneous :-)

[0]: http://www.iterm2.com
