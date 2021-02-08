---
layout: post
title: "The Testing ROI"
author: Javier García
category: testing
tags: testing, design, roi
---

A couple of days ago, in another conversation on testing our code, a colleague
mentioned something I found very interesting: **we should always be aware of
the ROI of our tests.**

As most of my readers know, I'm a big fan of testing. I find that it's a great
asset for designing our software. It provides a safety net so we can refactor
our code recklessly. It helps us break down big problems into smaller chunks.
It even makes pair programming more fun! You do the test, and I'll do the
implementation, and then swap.  All this IS truly amazing, specially when you
get into the habit of TDDing your code and it just ends up shifting your
mind completely and becoming a 6th sense.

But thinking about the ROI of tests provoked me something completely different.
It made me think: how many times have I invested 45 minutes into writting a
couple of tests, and then, after a couple of iterations on the code, had to
completely change them. Even further, after a couple of days and the code
evolving, had to obliterate them. Or, on a different scenario, how many times
have I invested time into testing certain boundaries made by HTTP, message
queues or databases, having to mock all that... and then at the end it didn't
really end paying off.

All this made me think of something more: software is just an ever-evolving
piece of work which is simply created as a tool, a solution to a problem.
I have found that when walking down the road of craftsmanship in software,
it's extremely easy to loose perspective of this. It's extremely easy to start
idolasing one's code, refactoring it for ever, testing every nook and cranny.
But that's not really the purpouse of software. I do strongly believe that the
values craftsmanship in our guild are something to fight for and that software
is indeed the product of our work, whose quality completely depends on our care
for it, however, we must never loose sight of **pragmatism**. Pragmatism is the
principle that brings equilibrium to the equation of creating software.

If the cost of our software breaking and us reparing it within a reasonable
timespan is infinitely smaller than the cost of developing a test suite for it
and evolving that suite as the features rapidly evolve, what makes more sense?
To fix the code as we grow it, or to invest in the suite? Of course, there is
not right answer here. It depends. However, I think it's important to take this
deeply into account. Recently I find myself thinking about this as I work on a
codebase that changes and evolves an absurdly fast pace. If it takes me twice
the time to write things and tomorrow I may end up rewritting the full thing
anyways... the ROI is shit.

And coming back to the ROI of our tests, at the end of the day, each new piece
of code we write requires different leveraging, but I realize how sometimes
when you discover a new tool, a new piece of knowledge, it's very easy to
happily apply to it to every scenario, just because you enjoy it, without
necessarily being the best tool for the job. And this can also be testing. Now,
I am, and probably will be for a very long time a fan of testing, but I think
embracing the thought that testing is not a silver bullet and that testing some
things might require more effort than the actual value the software provides is
a key player here.
