---
layout: post
title: "Migrating to Jekyll"
author: Javier Garcia
description: ""
category: blog
apprenticeship: true
tags: 8th-light, blog, jekyll
---

It's been a while since I wanted to centralize all my blogging into one place. The good thing is that this weekend I achieved my goal! For this, I used [Jekyll](https://jekyllrb.com/) & [Github Pages](https://pages.github.com/).

Before I had a personal tech blog in [Ghost](https://ghost.org/), another one dedicated to poetry in [Wordpress.com](https://wordpress.com/) and I very recently a opened a third one also in Wordpress.com for my apprenticeship at 8th Light.

One of the reasons why this situation wasn't working for me is because I hate when things don't have an order. The idea of having 3 different urls for what, at the end of the day, was "my writting" was killing me. Moreover, I didn't quite like Wordpress.com.

Wordpress.com is a very easy platform to learn. It takes no more than 5 minutes to get started. Maybe add an extra 10 minutes to customize your blog with a theme. The problem is the potential for customizability is very low. You can install a few plugins, but they always have to be from their own store – no developing your own. The same happens with themes. For some people this would be a solid pro – easy to get it running, but for me it was a con. For this, I prefered Ghost as a platform.

Ghost is a very nice platform. I actually like it more than Jekyll, but the problem it presents for me is that I have to host it somewhere like Heroku or Azure and this usually requires a little more work than commiting a Markdown file to a repo, like in Jekyll. If you want to make some changes to the styling, it works like any other server-side application: you make the changes in local, test, deploy to the hosting server and run ghost. The thing is, for me this felt a little overkill for a simple blog... so here came Jekyll.

With Jekyll I have found myself in the right spot; exactly where I wanted to be. Jekyll allows me to customize the blog 100%, as long as I keep it a static site. I can tamper with the HTML, the CSS, add some extra functionality with JS. On the other hand, to add a new post I simply have to write a Markdown file with the content and commit it to the blog's master branch. It's lovely and easy!

At the moment I'm using a theme I found online, [Julia](https://github.com/kuoa/julia). It's minimalistic, nice and simple. A good place to start, right? Nonetheless, as time passes, I would like to eventually develop my own theme. Plus it's a good way to get the basics of design with CSS. We'll see how it goes.

In the meanwhile, happy me blogging :)