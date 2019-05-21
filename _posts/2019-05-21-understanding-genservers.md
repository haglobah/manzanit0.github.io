---
layout: post
title: "Understanding Genservers"
author: Javier García
description: "Genservers under the hood"
category: elixir
apprenticeship: false
tags: elixir, otp, genserver, in-depth
---

If you're reading this, it probably is that you've had to use Elixir's `Genserver` behaviour already
but you're wondering how it works. For me, the first time I tried using it I got it completely wrong –
Instead of using it as a store of data I ended up spawning up a brand new genserver for each new
record I wanted to save. When I found out, I realised it was simply because I wasn't really
understanding what was going on under the hood.

## What is a Genserver?

A Genserver, according to the [hexdocs](https://hexdocs.pm/elixir/GenServer.html) is:

> A behaviour module for implementing the server of a client-server relation.
>
> A GenServer is a process like any other Elixir process and it can be used to keep state, execute
code asynchronously and so on. The advantage of using a generic server process (GenServer)
implemented using this module is that it will have a standard set of interface functions and
include functionality for tracing and error reporting. It will also fit into a supervision tree.

At the end of the day it's simply an abstraction for client-server behaviour: it allows us to define
a module of our own with a set of callbacks and we can call it from anywhere in our code base. It basically
serves two different purposes: first it allows us to execute pieces of code asynchronously, so we don't
necessarily block our main thread, and secondly it helps us save state.

In order to help us wrap our heads around this concept, we're going to build our own implementation of
a Genserver in Elixir. A really simple one!

## Processes 101

Before getting started, it's important to review how processes work and communicate in Elixir. In Elixir,
all code runs inside isolated processes. The basic mechanism for spawning those process is the `spawn/1`
function.

```Elixir
iex> spawn fn -> "Hello world!" end
#PID<0.117.0>
```

As you can see, `spawn/1` returns a process identifier (PID). Spawned processes usually execute the function
we have provided them with and then exit, so it's most likely dead. Take into account that wee can also spawn
processes with functions from already existing modules with `spawn/4`, but we'll check it out later in the examples.

Furthermore, in order for processes to comunicate Elixir provides us with the [`send/3`](https://hexdocs.pm/elixir/Process.html#send/3)
and [`receive/1`](https://hexdocs.pm/elixir/Kernel.SpecialForms.html#receive/1). `send/3` allows us to send a
message to any process, given that we know it's PID, and receive intercepts all the messages in the current process.
Check out this example:

```Elixir
iex> send self(), {:msg, "Hey!"}
{:hello, "world"}
iex> receive do
...>   {:msg, msg} -> msg
...>   _ -> "not a message!"
...> end
"Hey!"
```

Notice that we are sending a tuple to `self()`, which is a reference to the current process and once that is sent,
we intercept it by running a `receive/1` clause.

Now, taking all this into account we could probably write two simple modules which exchange messages in
the following way:

```Elixir
defmodule Person do
    def create, do: spawn(Person, :listen, [])

    def say(to, message), do: send(to, {:msg, message})

    def listen do
        receive do
            {:msg, msg} -> IO.puts("Said '#{msg}' to #{inspect self()}")
        end
        listen()
    end
end
```

And on `iex>`:
```Elixir
iex> marta = Person.create()
#PID<0.134.0>
iex> Person.say(marta, "Hey peep!")
Said 'Hey peep!' to #PID<0.167.0>
{:msg, "Hey peep!"}"
```

## Storing state in a stateless world

So far, in our journey to understand Genservers, we have learned about `spawn/1`, `send/3` and `receive/1`,
but, how does that take us any closer to understanding how genservers work?. Before jumping to the Genserver
behaviour we have still one more piece of the puzzle to unveil: **state**.

As you know, Elixir is a functional language which has now knowledge of state as is – we create modules, which
havew functions, and we give them data, which they spit out processed. But we don't have *instances* as we would
in C# or Java, instances that store state for us. Furthermore, when studying processes, as we spawn them, they die.
So how can we store state? The answer is recursion.

In order to be able to maintain some state in Elixir, the common pattern is for a process to recursively call itself
with the state it has. Take a look at the following example:

```Elixir
defmodule Counter do
    def init, do: loop(0)

    def loop(counter) do
        IO.puts counter
        loop(counter + 1)
    end
end
```

If you try to execute it...
```Elixir
iex> Console.init()
... (many numbers which I will not paste)
199943
199944
199945
199946
199947
199948
199949

BREAK: (a)bort (c)ontinue (p)roc info (i)nfo (l)oaded
       (v)ersion (k)ill (D)b-tables (d)istribution
^C
```

Yes, it starts printing all the numbers and won't stop until you stop the process, which you can do by pressing twice `Ctrl + C`.
Anyways, as you've seen, we have been able to store and change state with a recursive loop. That is the key to how
Genservers will hold state. Now, let us move forward to the real deal!

## A homemade Genserver

With all the tools we have gathered, we can now commence. Since I'm all about TDD, let's start with a test. Sticking to
the type of APIs Elixir provides us, I will want an `init/0` function which will spawn our server for us, and a `save/2`
function for saving the message to the server. This is my test:

```
defmodule StoreTest do
  use ExUnit.Case

  test "saves message" do
    pid = Store.init()
    response = Store.save(pid, "Hey!")

    assert {:ok, "message received!"} == response
  end
end
```

In order to make that test pass, we kind of do have to write a little bit of code. First we need the `init/0` function,
but we also need the loop we talked about which will be storing the state for us and a way to send back the response. If
you've tinkered around with Genservers a little, you will know that they allow the consumers to send both synchronous and
asynchronous messages via `call/2` and `cast/2`. In this case, we're trying to develop a function similar to `call/2` –
a function which waits for the server to create the response and return it.

First, the `init/0` function will use `spawn/4` which we mentioned at the beggining of the post, it will allow us to
pass it a function defined in one of our modules:

```Elixir
  def init() do
    spawn(__MODULE__, :loop, [[]])
  end
```

The third parameter are the args for the function, in this case we want it to be an empty list, so that explains the `[[]]`.

Continuing forward, we want our `save/2` function to send a message to the server and await the response, so
that should be fairly straightforward with `send/3` and `receive/1`:

``` Elixir
def save(pid, message) do
    send(pid, {:save, self(), message})

    receive do
        {:ok, response} -> {:ok, response}
    end
end
 ```

 Lastly, the loop. When we spawn the process, we're invoking a `loop/1` function which receives the
 state and is supposed to keep the wheel going. Since our `save/2` function sends a message to the
 server and awaits the response, we can kind of assume that the loop will be waiting for a message
 and then sending a response back.

 ```Elixir
def loop(state) do
    state =
      receive do
        {:save, from, msg} ->
          send(from, {:ok, "message received!"})
          [msg | state]
      end

    loop(state)
end
```

And if we put this all together...

```Elixir
defmodule Store do
  def init() do
    spawn(__MODULE__, :loop, [[]])
  end

  def save(pid, message) do
    send(pid, {:save, self(), message})

    receive do
      {:ok, response} -> {:ok, response}
    end
  end

  def loop(state) do
    state =
      receive do
        {:save, from, msg} ->
          send(from, {:ok, "message received!"})
          [msg | state]
      end

    loop(state)
  end
end
```

If we now run `mix test` we should have a nice green output. Next, we want a function
to retrieve all the messages we've stored. I start, as always, with the test:

```Elixir
test "retrieves all message" do
    pid = Store.init()
    Store.save(pid, "...world!")
    Store.save(pid, "Hello")

    response = Store.fetch(pid)

    assert {:ok, ["Hello", "...world!"]} == response
end
```

The good thing is, in this case, we just need to develop the `fetch/1` function. In this case, `fetch/1` will
look very similar to `save/2`, in the sense that it will send a message and expect a response. The core of the
logic will be coded in the loop – we need to make the server respond with all the data. Once we finish, our code
looks like this:

```Elixir
defmodule Store do
  def init() do
    spawn(__MODULE__, :loop, [[]])
  end

  def save(pid, message) do
    send(pid, {:save, self(), message})

    receive do
      {:ok, response} -> {:ok, response}
    end
  end

  def fetch(pid) do
    send(pid, {:fetch, self()})

    receive do
      {:ok, response} -> {:ok, response}
    end
  end

  def loop(state) do
    state =
      receive do
        {:save, from, msg} ->
          send(from, {:ok, "message received!"})
          [msg | state]
        {:fetch, from} ->
          send(from, {:ok, state})
          state
      end

    loop(state)
  end
end
```

## What about the async functions?

As I commented before, Genservers also have asynchronous handlers: `cast/2`. The reason why
I decided to only implement the synchronous handlers is because the async ones are simpler.
Our synchronous function looks like this:

```Elixir
def fetch(pid) do
    send(pid, {:fetch, self()})

    receive do
      {:ok, response} -> {:ok, response}
    end
end
```

Yet, if we want to make it async, we just have to delete the `receive/1` clause.

```Elixir
def fetch(pid) do
    send(pid, {:fetch, self()})
end
```

## Conclusions

After having read through the whole thing, Genservers don't look so dangerous anymore, do they? At the end
of the day they are, like our `Store` module, a simple wrapper around processes which communicate between
each other and provide us with a client-server architecture. Before finishing though, I will mention Agents.

[Agents](https://hexdocs.pm/elixir/Agent.html) are yet another abstraction provided to us by the Elixir core
team to make working with state easier. At the end of the day, they are simply a
[wrapper](https://github.com/elixir-lang/elixir/blob/master/lib/elixir/lib/agent.ex#L270) around Genservers
themselves, but they provide us with a much cleaner and easier API to use – we don't have to worry about
implementing the cast/call callbacks boilerplate. Next time you need to store state in your application,
give it a thought – Do you need a Genserver? Can you do it with a Task or an Agent instead? It's always
about simple code!
