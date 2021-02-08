---
layout: post
title: SOLID isn't just for object-oriented programming
author: Javier García
category: software design
tags: design, software, solid, functional
---

One of the first things we learn when we start to dive into object-oriented
programming are some of the principles that help us achieve cleaner and more
maintanable code: SOLID, Demeter's law... What not many people tell us is that
they're not necessarily object-oriented principles. I personally see them as
principles that apply to all kinds of software paradigms.

## Single Responsability Principle

The first solid principle is the Single Responsability Principle. It states
that any given class should have one, and only one, reason to change. If we
were to take this to a functional ground, we could translate it to _any given
function should have one, and only one, reason to change_, which I think is
pretty reasonable. But we could take it a step further and talk of [referential
transparency][0]. In a nutshell, a function is referentially transparent if we
can substitute it for it's output value and the program's behaviour will remain
identical. In other words, no side-effects take place, the function is pure.

While it might seem like a long shot to associate single responsability with
function purity, it actually has everything to do with it. Pure functions can
be seen as a simple calculation, or a set of operations applied to a piece of
data. As long as functions have one, and only one, responsability that
responsability maps to the calculation. The moment you start triggering
side-effects, those are _other operations_ happening within the same function.

Nonetheless, any application that is remotely serious will of course have to
deal with many kinds of interactions with the outside world. We're talking
HTTP, we're talking databases, logging, any kind of IO... you name it. It is
then unavoidable to have functions that deal with these, hence being impure.
But the same happens in object-oriented programming. At some point you have to
persist the blog post, once you've written it. **The important take here is to
know when and how to isolate that impurity in order to keep the [domain][1] of
the application pure**, following the single responsability principle.

## Open/Closed Principle

The OCP might seem like a slightly more complicated one to see through, but it
just depends on the right perspective. I'll talk about two things here: [higher
order functions][3] and [algebraic data types][4].

One of the most used mechanics in functional languages are the use of
higher-order functions. A HOF is simply a function that either takes a function
as an argument or returns a function as its result. This is possibly the second
most simple way of composing behaviour, of course after simply invoking the
function directly by it's reference. It enables us to dinamically inject
functions into other functions and modify the behaviour in runtime. In some
way, it's composition the functional way. Now, if we've already designed our
functions with the first principle in mind, **we can create new functions
composing the already existing ones in the most creative ways**, ocasionally
passing them as parameters too. Functions closed to modification, yet open to
extension.

The second topic that I find relevant is that of algebraic data types. At the
end of the day, an algebraic data type is a kind type made from combining
multiple types. A simple example is a tuple, since it can contain different
sets of types within. Another one are union types, like you can find in
Haskell, F# or Elm. If we start thinking of types in this way, we can then
start combining the existing types to obtain composites, hence obtaining this
kind of behaviour.

## Liskov Substitution Principle

Liskov's principle is one of my favourites. It basically states that any object
in a program should be replaceable with instances of subtypes without altering
it's correctnes. The simplest example is, if we have an `Iterable`, and we
substitute it for a `List`, our program should work just the same.

In the functional realm, it's slightly trickier to see the akin, but let's try.
I sometimes like to call the LSP as the `least surprise principle`. If I define
a function which takes a type `A`, does a set of operations over it, and
returns type `B`, then I would expect that if I feed it a type `T` which is a
subtype of `A`, it will still return me `B`, simply because `T` contains
everything that `A` does, plus a little more. Now, while it's true that not all
functional languages are typed, like Elixir or Clojure, others are, like
Haskell or F#. And it applies just the same. I like to see LSP as a rewording
of [contravariance in category theory][5]. **It's not about the paradigm, but
about types**.

## Interface Segregation Principle

I'll keep it simple: **feed your functions the smallest piece of data they
need**, nothing more, nothing else. If a function needs an an email to create a
user, then give it a string with the email, don't give it a full blown user
structure with email, name, phone, and siblings.

Of course, there are some scenarios where you may find it convenient to create
functions which actually take certain abstractions instead of the simple
primitives, and it's fair, but if you're going to do that, have a reason. Don't
do it because it's easier or it looks good. For me, an scenario where I like to
break this rule is when I'm creating a module of reducers, that is, a module of
functions which take the same structure and return said structure with some
modified content. I find this useful defining pipes of operations. But other
than that, the interface segregation principle is all about feeding the
smallest piece of data to our functions.

## Dependency Inversion Principle

Last but not least, we have the DIP, which says:

> High-level modules should not depend on low-level modules. Both should depend
> on abstractions. Abstractions should not depend on details. Details (concrete
> implementations) should depend on abstractions.

To me, it talks about depency injection. About function composition. Again,
about higher-order functions. It is with all these that we can obtain the
adequate abstraction in order to make our functions not depend on the wrong
details. In the following example, see how the `filter_customers/2` function
doesn't need to know the details of the filter being applied:

```elixir
def filter_customers(customers, filter) do
  Enum.filter(customers, &filter/1)
end
```

The `filter/1` function becomes the interface, and we program against it, not
against the concrete details.

## Wrapping it up

These are some ideas I've been gathering in the last weeks as I went back to
object-oriented programming for a specific project and starting comparing all
my previous experience in functional programming. As time progresses I cannot
but see how the line between functional and object oriented paradigms just
isn't that deep. It just requires that we open our eyes a little, and that we
go to the true core of the ideas underneath the words.

[0]: https://en.wikipedia.org/wiki/Referential_transparency
[1]: https://dddcommunity.org/learning-ddd/what_is_ddd/
[3]: https://en.wikipedia.org/wiki/Higher-order_function
[4]: https://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.28.6778
[5]: https://eli.thegreenplace.net/2018/covariance-and-contravariance-in-subtyping/
