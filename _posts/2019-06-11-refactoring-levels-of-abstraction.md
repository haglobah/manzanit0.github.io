---
layout: post
title: "Why keeping levels of abstraction matters"
author: Javier García
category: refactoring
tags: refactoring, abstractions, design
---

Most developers who have been in the profession for at least a couple of years have heard of software principles,
if not read about them or even learnt to apply some. One of the not so known, yet important, principles is the **Single Level of Abstraction principle**:

> Code within a given segment/block should be at the same level of abstraction.

### What are levels of abstraction?

A level of abstraction is but a metaphorical layer we add to our code to hide certain details of a subsystem.
This is mainly done to achieve more readable and refactorable code.

For example, let's say we have a part of our system that creates a list and then increments its values by ten:

```Elixir
def system(length) do
  1..length
  |> Enum.map fn x -> x + 10 end
end
```

If we were to add a layer of abstraction, it would then look like this:

```Elixir
def system(length) do
  length
  |> create_range
  |> increment_by_10
end

def create_range(size) do
  1..size
end

def increment_by_10(list) do
  Enum.map(list, fn x -> x + 10 end)
end
```

The main purpose of adding that metaphorical layer is to make the system much more readable and testable.
Even though the latter is lengthier, it takes less effort to read through than the first example. 
Most of the time the one factor that makes our code more readable is simply getting it closer to the way
humans express themselves. Like some say, *"Let the code explain himself."*

### Cultivating an expressive and ubiquitous language

It's true that sometimes certain parts of our code base are going to be very technical, especially when
the business requires plenty of calculations or has complex rules. Nonetheless, in order to come closer 
to the expressiveness we expect of our code, thinking about how we want to talk about it really helps.

One of the things I learned from reading *Domain-Driven Design*, by Eric Evans, is the importance of a rich
language-based model. If we're building an application for a restaurant, we would want to be asking the
chef questions like:

> "Should the menu be able to contain over 30 dishes?"

Rather than:

> "Should the length of our list which contains the dishes accept more than 30?"

If you really think about those two questions, they're talking about our model—about the abstractions
we choose, about the metaphor within our code. The restaurant's dishes might be stored in a list,
but we can definitely hide that implementation detail under a module that we can call `Menu`. So
instead of talking about lists of dishes, we talk about menus.

Of course, coming up with the appropriate metaphor and language to use in our code is no easy task.
Sometimes it can even take the whole length of the project, as we discover new corners of the business.
But the value of maintaining that domain-rich language throughout our code is invaluable. It's not
just much easier to understand what's written, but it enables the developer to maintain a conversation
with the business representatives without having to adapt the way he talks about what he's doing.

Furthermore, we want that language to be ubiquitous—that is, ever present throughout all layers of our
application. Once we come up with the correct metaphor, we don't want to be creating a different one for
the frontend, just because it's the frontend and it needs the data in a slightly different way.
If it's needed, extend the metaphor, but maintain the language we've already created.


### From words to tests

At this point, you might be asking yourself, *How does changing the language used throughout my application
make it more testable?* It's all about design. Just like TDD helps us improve our application's design
because it encourages us to decide how we want the API to look and behave before implementing the code,
choosing an appropriate metaphor helps the tests be more focused and behaviour-oriented.

We don't want to be checking if the title of our blog post is *capitalized* and *trimmed*, we want to
check if it's *formatted*. Just the same, we don't want to verify if the size of our list of dishes
is bigger than 30, we want to verify if our menu has over 30 dishes. As we adapt our code to meet our
domain-rich metaphor, our tests start to test the behaviour of the business and not just nitty-gritty details of the code.

### Other benefits of applying metaphors to our code

We've already commented two benefits we've obtained from applying an abstraction layer to our system:
readability and testability, but there is yet another just as powerful, **segregation of responsibilities**.

As we adapt our code to our metaphor of choice, we inevitably break it down into functions or classes,
depending on the paradigm we're coding in. The great thing about breaking down code is that if we do it
properly we can end with very small, single-responsibility forms, which lead to higher cohesion and lower coupling.

### How do we choose which abstractions to apply?

At the end of the day, it all comes down to what abstractions to apply. This greatly depends on the
application and the domain. I always like to ask myself, *is it expressive enough like this?*

The easiest scenario to identify is when we're doing low-level data transformations along with some
high-level operations. Say, for example:

```Elixir
def submit_blog_post(title, post, blog) do
  title
  |> String.downcase
  |> String.capitalize
  |> String.trim
  |> compose(post)
  |> publish(blog)
end
```

Could very easily be transformed to:

```Elixir
def submit_blog_post(title, post, blog) do
  title
  |> format_title
  |> compose(post)
  |> publish(blog)
end

def format_title(title) do
  title
  |> String.downcase
  |> String.capitalize
  |> String.trim
end
```

With such a trivial change we have achieved a lot. First of all, our `submit_blog_post` function reads fluently—we don't have to worry about the kind of transformations that are being applied to the title, just that it's being
formatted. As a developer, this allows me to focus on what the function is trying to achieve more than the actual
details of how. On the other hand, `format_title` comprehends all these transformations, allowing me to test them in an
isolated way without potential HTTP requests being made from the `publish call` getting in the way. Just like we said:
readability, testability, and expressiveness.

Unluckily, not all scenarios are as easily identifiable as the previous. Sometimes our code is tangled in a way
that would scare the fiercest of Spartans. In these cases, I encourage first, labeling the pieces of code, just
like we did in the previous example. As we slowly label pieces of code and extract them in functions, we are abstracting.
Maybe it's not the ideal level, but it's a starting point. Once we have our code nicely labelled in meaningfully
named functions, we can start trying to describe the business processes with a more appropriate metaphor and refactor.


### Wrapping up

The importance of picking a domain-rich metaphor for our code—one that allows us, as developers,
to express our thoughts in a language that is close to that of our business associates—is invaluable.
To maintain the levels of abstraction in our code shouldn't just be another tool in our toolbelt, but
one of the final goals for every project we work in. It's not something to approach as a single-day task,
but neither should it intimidate us. It's something that is achieved through long conversations and
constant iteration over our own work. Distilling the business and the model with which we will work is
something that comes by experimenting and brainstorming; but once we have developed it, the power of that
metaphor simply brings our code to a whole new level.

> *"Cultivate an expressive and ubiquitous language in your code. The rest just follows."*

*Originally posted at 
[8th Light's blog](https://8thlight.com/blog/javier-garc%C3%ADa/2019/06/11/refactoring-levels-of-abstraction.html)*