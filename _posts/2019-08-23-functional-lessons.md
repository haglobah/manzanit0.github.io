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
functional language functional. Today, nearly one year later, I'm pretty confident it's my favourite
paradigm. In this blog post my intention is to share this little part of my journey, and the different
things I learnt during it. Hopefully it may be of use to people who choose to start a journey like mine.

## It isn't always love at first sight

The first time I developed something in a functional language, I built a bot for Telegram in Clojure. That
was the birth of Londibot—a bot that would ping me every time the Victoria line had a disruption.
My mentor at that time gave me freedom to pick the project, and I thought it would be more fun to develop
something that I could afterward put to good use, in this case for my commute. The
language, though, he picked.

Dear reader, I'm not sure if you've ever tried any LISP dialect, but if you, like me, come from an
Object-Oriented world, the learning curve can get very steep. I remember trying to figure out how to loop over
lists to change the elements, trying to filter dictionaries... It's very different than what we're used to.
The first things I got acquainted with that really paid off were the classic `filter`, `map`, and `reduce`;
then came recursion to achieve loops, and many others. It was actually by learning these tools and giving
them enough time to sink in that I started to understand the real difference between imperative and declarative code,
and how the functional paradigm usually tends to favour more that declarative style. I could just say that learning a
functional language and its idioms truly changed my perspective of how to solve problems and, ultimately, of how to
express those solutions.

To summarize my first impressions, both the amount of parentheses that Clojure brought to the table (or any
LISP dialect for that sake), and the amount of new concepts I had to learn, made it very overwhelming. It took me
nearly a month until I started to enjoy the language a little bit more. The good thing is that at the end,
it does pay off.

## Keeping functions pure is crucial

One of the first mistakes I made early on in the project was not respecting the purity of the functions.
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

The _problem_ with the above is that all state is saved within our `CarPark` instance, and as we park or
unpark cars, that instance is modified. That's called a side effect. A side effect is any operation that
happens within a function that is collateral. Imagine a `sum` function that, apart from summing two
numbers and returning the result, also saved it to a database. The saving to the database is a side-effect.

I'm aware that I still haven't explained the concept of purity, but I wanted to talk about side-effects
first because it greatly helps illuminate purity. In a nutshell, *a function is pure when, for a same input,
it always has the same output and never has any side effects*. Printing a value to the log or simply changing
some mutable state within the application makes our function impure.

You might be wondering: *why is purity such a big deal though?*. Purity makes our functions little sandboxes.
It is by making pure functions that we can feel safe that when they change something it will only affect the
context within the actual function. And this brings us to immutability. As we make our functions pure, that means
that they only work with the parameters we provided them with AND they don't mutate them, but return a result
based on a copy of those parameters.

Now, that may sound confusing, but let's say that we rewrote the above code to look something like this:

```java
import java.util.ArrayList;
import java.util.List;
public class CarPark {
    private String status;
    private List<String> cars;
    public CarPark() {
        status = "Closed";
        cars = new ArrayList<>();
    }
    public CarPark(String status, List<String> cars) {
        this.status = status;
        this.cars = cars;
    }
    public CarPark open() {
        return new CarPark("Open", this.cars);
    }
    public CarPark close() {
        return new CarPark("Closed", this.cars);
    }
    public CarPark park(String car) {
        List<String> newList = new ArrayList<>(List.copyOf(this.cars));
        newList.add(car);
        return new CarPark(this.status, newList);
    }
    public CarPark unpark(String car) {
        List<String> newList = new ArrayList<>(List.copyOf(this.cars));
        newList.remove(car);
        return new CarPark(this.status, newList);
    }
}
```

Which when used would look like:

```java
CarPark park = new CarPark();
CarPark openPark = park.open();
CarPark parkWithPorsche = openPark.park("Porsche");
CarPark parkWithoutPorsche = openPark.unpark("Porsche");
CarPark finalPark = parkWithoutPorsche.close();
```

Now, of course that is not idiomatic Java, but if we look at it closely–every time a function is executed, it
provides us with a brand new instance with the attributes we expect. The major advantage of this is that if,
say for example, another process were to run one of those methods concurrently, it would not screw up the operations
running in other concurrent threads–each thread would be working with their own instances. Those functions are pure.

To wrap up all that I've just mentioned above, **pure functions are functions that don't create side effects and only
affect the context within the function itself, not outside**. As we keep our code pure, it is easier to test and overall
more predictable. Purity keeps the code simple, while impurity increases the complexity.

Now, coming back to functional languages, Elixir specifically solves a lot of those problems out of the box!
By default, all data structures are immutable in Elixir, which means that whenever they are altered, the runtime
will give back a complete brand new copy of the structure. This means that if we were to implement a `CarPark` module,
it would be:

```elixir
CarPark.new() # Creates a new struct
|> CarPark.open() # Creates a new struct with status = open
|> CarPark.park("porsche") # Creates a new struct with a porsche parked
|> CarPark.unpark("porsche") # ...
|> CarPark.close() # ...
```

The code for that would be:

```elixir
defmodule CarPark do
  defstruct[:status, :cars]
  def new(), do: %CarPark{status: :closed, cars: []}
  def open(%CarPark{} = park), do: %CarPark{park | status: :open}
  def closed(%CarPark{} = park), do: %CarPark{park | status: :closed}
  def park(%CarPark{cars: cars} = park, car_to_add) do
    %CarPark{park | cars: [car_to_add | cars]}
  end
  def unpark(%CarPark{cars: cars} = park, car_to_remove) do
    less_cars = Enum.filter(cars, fn car -> car == car_to_remove end)
    %CarPark{park | cars: less_cars}
  end
end
```

Now, like I just mentioned, most functional languages, like Elixir, come with immutability out of the box, and
that is already a fantastic asset to help us keep purity throughout our codebase. Nonetheless, that module would
stop being pure the moment I dump something along the lines of `Database.save(carpark)` in the functions.
And if we did that, things like testing would go from being trivial to being complicated and requiring a lot
of mental effort to keep track of the state of things.

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

The above piece of code is a beautiful example of how we can pipe multiple functions together, creating a highly readable function.

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
of effort required to follow either. They're doing the same thing, but the first requires much less cognitive effort.

A good rule of the thumb is to go with either horizontal functions or vertical functions,
and try to avoid mixing them as much as possible. There are many ways of achieving this, starting by
trying to keep the function heads consistent in the order of parameters. For example, always accepting
the struct to work with as the first parameter allows us to pipe functions much easier. Take
a look at the `upload/2` function above. By changing the order of the parameters we can either pipe it
or not. A good example of this is the [`Enum`](https://github.com/elixir-lang/elixir/blob/master/lib/elixir/lib/enum.ex)
module in Elixir. Just notice that for the Enum module to be pipeable it always accepts the enum as the first parameter
and always returns a list!

```elixir
[1, 2, 3, 4]
|> Enum.reverse()
|> Enum.map(fn x -> x*x end)
|> Enum.filter(fn -> x > 5 end)
|> Enum.reduce(fn x, acc -> x + acc end) # returns 25
```

For me, realizing this was huge. I remember when I developed my first modules with Elixir, I didn't care much
about the order of the parameters. After all, it's not something you worry about that much in C# or Java,
so most of my functions tended to read horizontally with an occasional vertical bit. When I started thinking
about it, my thought process changed, and with it the readability of my code.

## Reducers, gotta love 'em

Following up on the order of parameters, come reducers. At the end of the day *a reducer is simply a function
that accepts some data, applies some transformations to it, and returns the modified version of the data*. You'll
probably be thinking that it's what all functions do, but the highlight here is that reducers don't return the
specific piece of data transformed, but the whole data WITH the transformed bits.

Say for example the `String.capitalize/1` function:

```elixir
iex> String.capitalize("abcd")
"Abcd"
```

It takes a string, capitalizes it, and returns the new string. When the function is applied, it gives us a new
string with the `A` capitalized, not just the capital `A`. A different reducer that uses structs but achieves a
similar purpose could look like:

```elixir
def capitalize(%SomeStruct{value: value} = s) do
  %SomeStruct{s | value: String.capitalize(s)}
end
```

Instead of returning simply the new `value`, we return the complete structure, like for example `IO.inspect/1`.
Some different reducers operating on the same structure could be:

```elixir
def hash(%SomeStruct{value: value} = s) do
  hashed = :crypto.hash(:sha, value) |> Base.encode16
  %SomeStruct{s | value: hashed}
end
def downcase(%SomeStruct{value: value} = s) do
  %SomeStruct{s | value: String.downcase(s)}
end
```

Now, if we combine all these together, the value really sticks out. They allow us to do things like:

```elixir
iex> %SomeStruct{value: "javier"}
     |> capitalize()
     |> hash()
     |> downcase()
%SomeStruct{value: "7f3d0970ec0e336aa08a9e14d4d88e79131e0065"}
```

That, my friends, is the magic of reducers. By applying transformations to our data structures and always
making sure we return that same data structure, we can endlessly pipe our reducers, thus achieving the vertical
functions we spoke about earlier.

Having the core business logic of our application modeled with reducers, as long as it makes sense, also
makes it very easy to test:

```elixir
def assert_result(%SomeStruct{value: v} = s, expected) do
    assert v == expected
    s
end
test "complex hashing works!" do
    %SomeStruct{value: "javier"}
    |> capitalize()
    |> assert_result("Javier")
    |> hash()
    |> assert_result("7F3D0970EC0E336AA08A9E14D4D88E79131E0065")
    |> downcase()
    |> assert_result("7f3d0970ec0e336aa08a9e14d4d88e79131e0065")
end
```

In my opinion, that is simply a beautiful test. I wish all my tests read in that clean and simple way.
Reducers empower us to write better, cleaner code. It's key though that the order of the parameters are
consistent (so we can pipe our functions), and that they are also pure (so all the side effects are isolated
in the boundaries of our application).

## Encapsulate data with functions

Another lesson I learnt while coding with Clojure, early in my functional journey, was how to encapsulate
data. One of the core pillars of OOP is encapsulation—keep your data private and your behaviour public.
That way if some day you decide to modify the underlying implementation of your behaviour, it will never
affect the consumers of your API.

In FP sometimes it can be slightly more challenging, or maybe just different, especially when the language
doesn't expose the concept of complex typing. I found out that by encapsulating data with functions, I was
able to make my applications much more robust—consumers would have a consistent API with which to interact.
In Elixir we have structs that solve that problem for us; but in Clojure, we don't. So instead of passing
an anonymous map around and accessing its keys, we could do something like:

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
our consumers how we want them to use our API—we're telling them, **"Don't use that property, because
tomorrow it might not be there!"**.

## Wrapping up

Coming to an end, all I can say is... stay curious, keep learning, question yourself. Functional purity,
encapsulation, and readability are all solutions to problems that emerged only after the code just didn't click.
All of this is just a quick summary of the different things I have learnt after having screwed up on many
occasions. [Londibot](https://github.com/Manzanit0/londibot) started as a Clojure bot when I was just
starting to learn functional programming, and it was thanks to poor design choices and bad decisions that
I kept reading and searching for better solutions to the problems that arose in my codebase. That's how all
this came up.
