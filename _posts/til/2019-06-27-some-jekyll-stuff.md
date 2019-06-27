---
layout: post
title: "The difference between nested posts in Jekyll"
author: Javier Garcia
category: jekyll
tags: jekyll, blog
---

Today I learnt that the only difference between a post saved under `_posts/`
and another in `_posts/til/` is the path. This has enabled be to use the
following logic in Liquid for the blog:

```Liquid
for post in site.posts
  if post.path contains 'til'
    <!-- Render TIL post title and date -->
  endif
endfor
```

A post saved under `_post/til/` would have a `post.path` of
`_post/til/2019-01-01-post-name.md`.

On another note, I also learnt that you can't add valid Liquid code to a post
because it renders. That's why the above snippet doesn't have the
_parenthesis - percentage_ Liquid syntax.
