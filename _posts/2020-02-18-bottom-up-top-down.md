---
layout: post
title: Notes on designing with tests
author: Javier García
category: software design
tags: testing, design, software
---

This morning I had the most interesting conversation at work with my colleague
Marco. I was ranting about how every time I decide to write tests for a piece
of software I've written I end up finding that it's way too tangled. It's thanks
to those tests that I am able to clean through my code like slashing with a
machete through the jungle, but I should've written them before to avoid much of
the complexity in the first place.

Then, in the middle of my ranting, he started laughing and mentioned something
that totally struck me:

> **Our design flaws don't really happen because we don't test first, but
> because we keep oscilating between a top-down and a bottom-up design approach**.

In systemics, both top-down and bottom-up are strategies for processing
information, management and ordering. When talking about software, it usually
refers to designing the system starting with the smallest granular components,
and then composing them together to obtain the system, or on the other hand,
starting by building the system in an abstract way and slowly building the
small components as we need them.

I hadn't really realised that, just like Marco mentioned, when you constantly
toggle from one approach to the other without being completely aware, you end
up with the worst from both approaches: a component that merely needs a single
piece of data ends up working with a huge blob which the system provides and
other parts of the system which need to be more generic end up having too many
details. To illustrate my point, say this function:

```elixir
def do_something_important(%WebContext{property: property}) do
  property
  |> process()
  |> some_extra_processing()
  |> format_result()
end
```

Could be changed to greatly simplify the cognitive load and the testability to:

```elixir
def do_something_important(property) do
  # ...
end
```

There are many arguments which could be used in favour or against, but at the
end of the day, the first snippet requires me to set up a `WebContext` in the
test, and the latter doesn't.  Simple.

Now, coming back to the topic of testing, I rediscovered, once again, that by
testing our programs we are able to achieve that uniform, simple interface that
otherwise would be screwed.  Before having this conversation, I found myself
passing down part of the stack dependencies to the web framework, the database
and the json parser. And it was thanks to trying to write a test that I
realised how convoluted it was.

I'm not saying that we can only write simple, clear and concise code when we
test first, like Marco said, if we are rigorous in our train of thought and
keep in mind if we're following a top-down or a bottom-down approach, it's also
possible to properly assign responsabilities, but it is true that the actual
effort of trying to test our code forces us to seggregate ideas and concepts in
a reasonable way, unless we want to be jumping hoops in our tests. But that's
an easy to spot smell.

These past months working in Rekki have been a challenge in regards to testing.
Mostly because some colleagues have different opinions like _tests take too
much time, we need to move fast_, or _maintaining tests makes changing code too
laborious_. And I mean a challenge in a good way, I love when people question
my ideas and principles, but today I was able to come back to the original
reason why I loved tests in the first place. Tests are the first real consumer
of my code, and they help me reason about my actions in a simpler way.

Now, for those interested in the top-down, bottom-up topic, Djistra wrote a
full monologue on it [here][0]. Feel free to give it a read, it's worth every
word.

[0]: https://www.cs.utexas.edu/users/EWD/ewd02xx/EWD249.PDF
