---
layout: post
title: "Functional lessons learnt"
author: Javier García
description: "Things I learnt while programming in functional languages"
category: functional programming
tags: functional, elixir, clojure
---

I still remember the time I spiked my first application in a functional language—it was Autumn 2018,
I had come to 8th Light to start my journey as a crafter, and I didn't even know what really made a
functional language functional. Today, nearly one year afterwards, I'm pretty confident it's my favourite
paradigm. In this blog post my intention is to share this little part of my journey, and the different
things I learnt during it. Hopefully it may be of use to people which choose to start a journey like mine.

## It isn't always love at first sight

The first time I developed something in a functional language, it was a bot for Telegram, in Clojure. That
was the birth of Londibot—a bot that would ping me every time the Victoria line had a disruption.
My mentor at that time gave me freedom to pick the project, and I thought it would be more fun to develop
something that I could afterwards put to good use, in this case for my commute. The
language, though, he picked.

Dear reader, I'm not sure if you've every tried any Lisp dialect, but if you, like me, come from an
Object Oriented world, the learning curve can get reeeal steep. I remember trying to figure out how to loop over
lists to change the elements, trying to filter maps... It's very different than what we're mostly used to.
The first things I got acquainted with that really paid off were the classic `filter/2`, `map/2`, and
`reduce/3`. Afterwards, recursion, immutability... We'll go through some of these in more detail in
a minute.

To summarize my first impressions, both the amount of parentheses that Clojure brought to the table (or any
lisp dialect for that sake), and the amount of new concepts I had to learn, made it very
complicated. It took me nearly a month until I started to enjoy the language a little bit more. The good
thing is that at the end, it does pay off.

## Keeping functions pure is crucial

One of the first mistakes I made early on in the project was to not respect the purity of the functions.
In most OO languages, it is standard practice to create classes that are then instantiated into objects,
which we then modify on demand. Say for example that you were to create a car parking app.
You would probably think of the car park like the following:

```java
var carPark = new CarPark();
var porsche = new Car("Porsche");

carPark.open();
carPark.park(porsche);
carPark.unpark(porsche);
carPark.close();
```

The *problem* with the above is that all state is saved within our `CarPark` instance, and as we park or
unpark cars, that instance is modified. That's called a side-effect. A side-effect is any operation that
happens within a function that is collateral. Imagine a `sum/2` function that, apart from summing two
numbers and returning the result, also saved it to a database. The saving to the database is a side-effect.

I'm aware that I still haven't explained the concept of purity, but I wanted to talk about side-effects
first because it greatly helps understand purity. In a nutshell, a function is pure when it has no
side-effects. Easy, right? Making functions pure isn't that easy if you're not used to it, though. Printing
a value to the log, changing a global variable, or simply changing some mutable state within the application
makes our function impure.

You might be wondering: *why is purity such a big deal though?*. That's the lesson I learnt the hard way.
By the the time I realised, I had a whole layer that was scheduling cron jobs, saving records to a database,
and interacting with the web—a bunch of bugs, and no easy way to create tests to fix them.

As we keep our code (especially our domain logic) pure, it is easier to test and overall more predictable.
Purity keeps the code simple, while impurity increases the complexity. Let's check out an example to
illustrate it.

Say for example that you want to calculate the sum of two numbers and log the operation made in a database.
If go for the fastest possible solution, also impure, it would look somewhat like this:

```elixir
defmodule Maths do
  @repository Application.get_env(:my_app, :repository)

    def sum(a, b) do
        result = a + b
        @repository.insert(sum)
        result 
    end
end
```

The problem is that to test that solution, I have to mock the repository; and whenever I use `Maths`
somewhere else, I would have to do the whole wiring again. On the other hand, if I separate it in two
layers (the domain logic and the consumer, which will always be impure), I don't have to worry about
complicated testing in my domain.

```elixir
defmodule Maths do
    def sum(a, b), do: a + b
end

defmodule MathsService do
    @repository Application.get_env(:my_app, :repository)

    def sum(a, b) do
        result = Maths.sum(a, b)
        @repository.insert(sum)
        result 
    end
end
```

Initially it might seem complicated to understand the benefit of doing the above, since it seems that we're just wrapping our code in more functions that aren't adding any extra functionality, but having all the domain logic
pure means we can test it without having to worry about persistance, HTTP, logging, or any other complex issues.
Eventually we will mock certain modules and test the DB layer, and the HTTP wiring... but without having
to worry if the core of the app, the business logic, is wrong.

## Make functions read only horizontal or vertical

Changing to a lighter topic, I also wanted to talk about function readability. Most functional languages
include some way of piping code—the pipe operator `|>` in Elixir or the threading macros `->` and `->>`
in Clojure. This is a huge tool for making our code very readable, as it allows us to develop whole pipelines
of transformations to our data in a very clean way. Say for example:

```elixir
load()
|> format_head()
|> format_body()
|> convert_to_pdf()
|> upload(:google-drive)
```

The above piece of code is a beautiful example of how we can pipe multiple functions together, giving place
to a highly readable function.

On the other hand, if we start mixing our style...

```elixir
{:ok, document} = load()

pdf =
    document
    |> format_head()
    |> format_body()
    |> convert_to_pdf()

upload(:google-drive, pdf)
```

The above is a simple example, but try to read both functions a couple of times and think about the amount
of effort required to follow either. They're doing the same, but the first requires much less cognitive effort.

A good rule of the thumb is to go with either horizontal functions, or vertical functions,
but try to avoid mixing them as much as possible. There are many ways of achieving this, starting by
trying to keep the function heads consistent in the order of parameters. For example, accepting
always the struct to work with as the first parameter allows us to pipe functions much easier. Take
a look at the `upload/2` function above. By changing the order of the parameters we can either pipe it
or not.

For me, realizing this was huge. I remember the first modules I developed with Elixir I didn't care much
about the order of the parameters. After all, it's not something you worry about that much in C# or Java,
so most of my functions tended to read horizontally with an occasional vertical bit. When I started thinking
about it, my thought process changed, and with it the readability of my code.

## Reducers, gotta love 'em

Following up on the order of parameters, come reducers. At the end of the day *a reducer is simply a function
that accepts some data, applies a function to it, and returns the modified version of the data*.
Say for example the `Enum.reduce/3` function:

```elixir
Enum.reduce([1, 2, 3], 0, fn x, acc -> x + acc end)
```

It's taking a list of data, iterating over it, summing each number, and returning the final result.
On the other hand, a different reducer that uses structs but achieves a similar purpose could look like:

```elixir
def accumulate(%Computation{result: r, to_process: n}) do
    result = Enum.reduce(n, r, fn x, acc -> x + acc end)
    %Computation{result: result, to_process:[]}
end
```

The main difference is that the latter gets the data from the struct. The cool thing about it is that it allows
us to do things like:

```elixir
iex> %Computation{result: 0, to_process: [1, 2, 3]}
     |> accumulate()
     |> Map.put(:to_process, [4, 5, 6])
     |> accumulate()

21
```

That, my friends, is the magic of reducers. By applying transformations to our data structures and always
making sure we return that same data structure, we can endlessly pipe our reducers, thus achieving the vertical
functions we spoke about earlier.

Having the core business logic of our application modeled with reducers, as long as it makes sense, also
makes it very easy to test it:

```elixir
def assert_result(%Computation{result: r} = c, expected) do
    assert r == expected
    c
end

test "computation works!" do
    %Computation{result: 0, to_process: [1, 2, 3]}
    |> accumulate()
    |> assert_result(6)
    |> Map.put(:to_process, [4, 5, 6])
    |> accumulate()
    |> assert_result(21)
end
```

In my opinion, that is simply a beautiful test. I wish all my tests read in that clean and simple way.
Reducers empower us to write better, cleaner code. It's key though that the order of the parameters are
consistent (so we can pipe our functions), and that they are also pure (so all the side effects are isolated
in the boundaries of our application).

## Encapsulate data with functions

Another lesson I learnt while coding with Clojure, early in my functional journey, was how to encapsulate
data. One of the core pillars of OOP is encapsulation—keep your data private and your behaviour public.
That way if someday you decide to modify the underlying implementation of your behaviour, it will never
affect the consumers of your API.

In FP sometimes it can be slightly more challenging, or maybe just different, especially when the language
doesn't expose the concept of complex typing. I found out that by encapsulating data with functions, I was
able to make my applications much more robust—give them a consistent API. In Elixir we have structs, which
solves that problem for us; but in Clojure, we don't. So instead of passing an anonymous map around and
accessing its keys, we could do something like:

```clojure
(defn new-job [userid cronexpression service]
  {:userid userid :cronexpression cronexpression :service service})

(defn get-id [job]
  (:id job))

(defn get-cron-expr [job]
  (:cronexpression job))

(defn get-user-id [job]
  (:userid job))
```

Thus allowing consumers to access the data like:

```clojure
(defn some-function [job]
  (-> job
      (job/get-user-id)
      (do-something)
      (do-something-else)))
```

By encapsulating our data sometimes with structs (but when not possible, with functions), we're telling
our consumers how we want them to use our API—we're telling them **"Don't use that property because
tomorrow it might not be there!"**.

## Rethinking state

The last thing I want to talk about is state. Mutable shared state.

Again, since I came from an OO world, I was used to creating instances and modifying their properties, and
doing endless things with them. But in some languages... that's just not possible. Elixir's data structures
are immutable—if you change a map, it will generate you a new map with the modified bit, and the old one will
remain the same.

So the question arises—How can I store state in my application? Say I'm spiking a solution and I don't want
to wire a database, but simply manage some data in-memory. Or say that my application actually requires some
in-memory state! How can you do that? The answer I have found is that each language solves the problem in their
own personal way. Elixir brings the concept of
[light-weight processes](https://elixir-lang.org/getting-started/mix-otp/agent.html)
to the table, whereas Clojure uses [atoms](https://clojure.org/reference/atoms) and [refs](https://clojure.org/reference/refs).
At the end of the day, depending on which language you're using, you'll have to find out which is
the idiomatic way.

Nonetheless, regardless of how each language solves the problem of managing mutable shared state, I have found
in my experience that the less, the better, and as we move all our state towards a database and drive it away
from the application, the problems we have to solve down the road become simpler, and less tangled. Managing
mutable shared state is difficult, so it's always a good bet to keep it isolated in the database.

## Wrapping up

Coming to an end, all I can say is... stay curious, keep learning, question yourself. All of this is just a quick
summary of the different things I have learnt after having screwed up on many occasions. Londibot
started as a Clojure bot, but as technical debt piled up due to the poor decisions I made, I started from
scratch in Elixir and became a happier man. But it wasn't because I changed stack (maybe a little), but because
I was able to improve all the design mistakes I had made before.

Stay curious,

Javier