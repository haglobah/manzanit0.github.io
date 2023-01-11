---
layout: post
title: "Vim: search and replace"
author: Javier Garcia
category: vim
tags: vim
---

Disclaimer, you can get away with a simple `sed`:

```sh
sed -i 's/pattern/replacement/' <files>
```

## Using vim

The trick goes as follows:

1. Populate the quickfix list with all the ocurrences you want to change
2. Leverage `cfdo` to replace

To populate the quickfix list you can use `vimgrep`, `Ggrep` from
[vim-fugitive](https://github.com/tpope/vim-fugitive), or anything else that
works for you. It doesn't really matter. For instance, let's go with `vimgrep`
since it's built in:

```sh
:vimgrep MyFunctionName **/**
```

Now check the ocurrences by opening your quickfix list: `:copen`. If you're ok
with that, then simply apply the change with:

```sh
:cfdo s/<search term>/<replace term>/g
```

It's as simple as that.

## Baking it into your workflow

One of the nice things of leveraging `cfdo` is that it's so baked into vim,
that it just plays nicely with other workflows. Say you're using using Neovim
and the built in LSP, and you're **not** using Telescope or any other fancy
plugins, but the actual built-in functionality which leverage quickfix/location
lists. This means that we every time you check the references of a function or
the uses of a module, you can just run `cfdo` and rename the whole thing,
without any other commands needed! 

This is particularly useful when the LSP doesn't provide renaming functionality
(although most do nowawsays ¯\_(ツ)_/¯).
