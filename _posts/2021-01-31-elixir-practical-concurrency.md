---
layout: post
title: "Elixir: Practical Concurrency Cookbook"
author: Javier Garcia
category: elixir
tags: elixir, concurrency, otp
---

While the Erlang runtime is known for being a highly concurrent platform to
which Elixir compiles, most of us end up solving the same problems in our daily
jobs. We use Phoenix to bootstrap our web applications, write JSON APIs and
sprinkle our pages with some javascript. And although it is true that simply by
using Phoenix we're already getting for free the whole concurrency model, the
majority of the features we develop for our products don't leverage all that
often distributed Erlang, complex supervision trees or fleets of GenServers.

In this post my intention is to highlight some common scenarios where we can
leverage Erlang's concurrency model as well as Elixir's abstractions to build
better, faster and more secure software. Think of it like... A cookbook.

## Fire and forget

The easiest problem we may have to solve is how to do a "fire and forget"
computation. In other words, how can we tell our system to execute some code
asynchronously and not care about when it finishes, nor the result. To do this
Elixir provides with the `Task` abstraction.

Tasks are processes meant to run a single action within their life-cycle. It
can be a long lived operation, like batch processing of records, or a short
one, like sending a Slack notification.

Usually, to run a one-off task the easiest thing is to use `Task.start/1`.
However, here is the first tip: don't do that. The best way to run a one-off
action in Elixir is to spawn the tasks under a supervision tree, with
`Task.Supervisor.start_child/2`.

The main reason why it's preferable to run tasks under their own supervision
tree is to allow for a proper clean-up of processes. When a Supervisor is taken
down, so are all its children, which allows you take down the application
cleanly.

This doesn't mean that you want your tasks to be restarted. In fact, the
default strategy for the `Task.Supervisor` is `:temporary`, which means they're
never restarted. It's just a means to avoid dangling processes if things go
weird at some point. And they always do.

Here is an example:

```elixir
defmodule FireAndForgetExample.Application do
  use Application

  def start(_type, _args) do
    children = [
      # Start the supervision tree under the OTP Application.
      {Task.Supervisor, name: FireAndForgetExample.TaskSupervisor}
    ]

    Supervisor.start_link(children, strategy: :one_for_one)
  end
end

defmodule FireAndForgetExample.OtherModule do
  def process_event(event) do
    # Start the task under the supervision tree.
    Task.Supervisor.start_child(FireAndForgetExample.TaskSupervisor, fn ->
      send_slack_notification("Hey! We got an event!")
    end)

    event
    |> do_something()
    |> do_something_else()
  end
end
```

## Fan-in/fan-out

However, what if we do care about the results? Sometimes it's useful to run a
certain operation and forget about it, but most times we actually do want to do
something with its result.

If the problem we have at hand is one which consists of multiple operations
which you can run asynchronously because they don't depend on each other, like
for example uploading a bunch of documents to S3, or sending a batch of emails
to different people, the easiest solution is to implement a _fan-in/fan-out_
strategy.

This can be done by using tasks too. We can do it naively without using a
Supervisor or we can spin them up under a Supervisor like we've mentioned
before. I always recommend using a Supervisor in code that is going to be
shipped to production, however, for the sake of simplicity, let's see an
example without it:

```elixir
defmodule FanInFanOutExample do
  def send_notifications(notifications) do
    notifications
    # Spin a task per element
    |> Enum.map(&Task.async(fn -> send_single_notification(&1) end)
    # Await all of them
    |> Enum.map(&Task.await/1)
  end

  def send_single_notification(notification) do
    # ...
  end
end
```

The good thing about this approach is that it will only take as long as the
longest task, and by the time the function has finished we will have a list
with all the results. In Elixir 1.11 there's also `Task.await_many/2`, which
under the hood it does a little bit more than a simple iteration and an await,
but in the end does get us to the same place.

## Scheduling work

A different kind of problem we might come across is "\_How can we run some work
periodically every N minutes/seconds/etc.". This is fairly simple leveraging a
different abstraction available in Elixir but ultimately in OTP: the
[GenServer](https://hexdocs.pm/elixir/GenServer.html).

GenServer is short for a "Generic Server". It's basically a process that can
receive messages and allows us to specify callbacks so it does different things
with those messages. I won't go very much into detail about GenServers because
[I already wrote about them a while back](https://manzanit0.github.io/elixir/2019/05/21/understanding-genservers.html).

So, how do we do the scheduling with a GenServer? Simple: by leveraging the
`handle_info/2` callback and `Process.send_after/3`. In other words, we will
add a callback to the GenServer which does the work and then schedule the
message with `Process.send_after/3`. Lastly, to make sure it runs again after
some time, we make sure to call `Process.send_after/3` again before the
callback returns. Let's see how it looks:

```elixir
defmodule SchedulingExample do
  use GenServer

  @default_minutes 3

  def start_link(args \\ []) do
    GenServer.start_link(__MODULE__, to_map(args))
  end

  defp to_map(args) do
    %{
      minutes: Keyword.get(args, :minutes, @default_minutes),
      forever: Keyword.get(args, :forever, true),
    }
  end

  def init(%{minutes: minutes} = state) do
    schedule_work(minutes)
    {:ok, state}
  end

  def handle_info(:work, %{minutes: minutes, forever: forever} = state) do
    # Do my work here ...

    if forever do
      schedule_work(minutes)
    end

    {:noreply, state}
  end

  defp schedule_work(minutes) do
    milliseconds = to_milliseconds(minutes)
    Process.send_after(self(), :work, milliseconds)
  end

  defp to_milliseconds(minutes) do
    minutes
    |> :timer.minutes()
    |> Kernel.trunc()
  end
end
```

Also, when leveraging GenServers and other abstractions over processes, it's
usually a good call to keep the server module with as little business logic as
possible. In this particular case if the GenServer can simply call a function
from a module, we're golden. This way we can decouple completely process
management from our business, allowing for easier testing... And an easier
time.

Lastly, we probably want to spawn these kind of worker processes under a
Supervisor too, but probably with a different strategy, like a `:one_for_one`,
so they're restarted in case they crash.

### Running it at specific times

In some cases we might want to run our code at specific times. Not necessarily
_every 3 minutes_, but _every day at 08.00AM_. While this is perfectly
achievable leveraging the same tools, I'll be pragmatic and recommend
[Quantum](https://github.com/quantum-elixir/quantum-core). It allows you to
schedule the execution of functions with a cron syntax and takes away all the
complexity of managing processes. It's a seasoned library, widely-adopted
within the community, very lightweight... And extremely simple.

## Caching data that's accessed very often

Occasionally you will find yourself with an endpoint that that makes queries
that take too long, or a process which consistently has to crunch a lot of data
and provides a bad user experience. In these situations _sometimes_ caching the
results may make sense. Sometimes it might make more sense to spend a few hours
tweaking the queries themselves or redesigning the solution, but sometimes
caching might make sense. Let's talk about when it does.

### The simplest approach: Agents

If you're already experienced with Elixir or Erlang, you'll know that its data
structures are immutable, however, it has its own way to work with shared
mutable state: processes. In order to save some state, access it and change it,
we can do so with a process and in many different ways.

The most simple solution for holding some state is creating an
[Agent](https://hexdocs.pm/elixir/Agent.html). Agents are the simplest possible
abstraction around state, and sometimes, if what we need is precisely a simple
solution without too many batteries, it might actually be the best option. One
of the good things about Agents is that it is a _single_ process, which means
that many concurrent clients will get their share of the Agent sequentially,
which means you don't have to worry about race conditions. On the other hand,
that can also be a bad thing if it starts becoming a bottleneck.

### An ETS based approach

Other times, if the Agent doesn't cut it for you, you might something faster.
In these cases [ETS](https://elixir-lang.org/getting-started/mix-otp/ets.html)
might be a good option. The good thing about ETS is that it will always be
faster because it doesn't go through the Erlang Scheduler, furthermore it also
supports concurrent reads and writes, which the Agent does not. However, it's a
bit more limited when you want to do atomic operations. Overall it's very well
suited for a simple shared key/value store, but if it's better suited or not
for your specific problem, that's up to you. An naive approach could look
something like the below:

```elixir
defmodule EtsCacheExample do
  def init!(seed, table_name) when is_atom(table_name) do
    case :ets.info(table_name) do
      :undefined ->
        :ets.new(table_name, [:set, :public, :named_table])

      _ ->
        raise "ETS table with name #{table_name(pool_name)} already exists."
    end

    add(seed, pool_name)
  end

  def teardown(table_name) do
    :ets.delete(table_name)
  end

  def add(value, table_name) do
    :ets.insert_new(table_name, {value})
  end

  def exists?(value, pool_name) do
    case :ets.lookup(table_name, value) do
      [] -> false
      _ -> true
    end
  end

  def retrieve_all(table_name) do
    table_name
    |> :ets.match({:"$1"})
    |> List.flatten()
  end
end
```

### One per user: GenServers

The third option I'm going to mention are GenServers. Most of the times either
the Agent or ETS should be enough, however in some cases it might make sense to
provide each API user with its own little cache. A good reason could be because
we need to provide with a certain atomicity for the read/writes and an Agent
would be a bottleneck. One of the strengths of GenServers is that they allow us
to spin one up per user very easily, so they don't become a bottleneck where
the Agent does.

### Some last thoughts on caching

However, while Elixir does provider the necessary abstractions to make caching
easy, my recommendation on this one is usually to lean on the community's
shoulders. Saša Jurić for example wrote a while back
[ConCache](https://github.com/sasa1977/con_cache) which does exactly this, but
there are many others out there. The good thing about not implementing one it's
own is that there are many edge cases when dealing with concurrency and it's
easy to get it wrong the first few times. Like they say, the two hardest
problems in computer science are naming and caching.

## Wrapping up

With all this, I hope to have shed some light in some potential solutions to
some of your maybe problems. Like all problems in IT, every solution sometimes
makes sense and sometimes it does not, but at the end of the day I believe that
these are all staple techniques within any Elixir or Erlang developer. Elixir
makes it so easy to work with concurrent code that most problems can be
tack;led without the need to bring a third-party library. There are of course
more than could be mentioned, like using
[`gen_statem`](https://erlang.org/doc/man/gen_statem.html) for state machines,
or worker pools for throttling work, but I'll leave those for another day.

Oh, and one last thing, don't forget to supervise your processes... It's
fault-tolerance for free :)

_Originally posted at
[Functional Works' blog](https://functional.works-hub.com/learn/elixir-practical-concurrency-3794f)_

## Further reading

Some posts and threads in case you want to read more on the topic:

1. [Discussion about uses for Agent Processes](https://elixirforum.com/t/discussion-about-uses-for-agent-processes/4214/37)
2. [Caching: ETS, Mnesia, Redis](https://elixirforum.com/t/caching-ets-mnesia-redis/6323/16)
3. [Data caching: Agents or ETS](https://elixirforum.com/t/data-caching-agents-or-ets/1614/18)
4. [Make Phoenix Even Faster with a GenServer-backed Key Value Store](https://thoughtbot.com/blog/make-phoenix-even-faster-with-a-genserver-backed-key-value-store)
5. [Best practice for doing fanout async calls](https://elixirforum.com/t/best-practice-for-doing-fanout-async-calls/24672)
6. [Optimising a crawler](https://manzanit0.github.io/elixir/2020/09/09/optimising-crawler.html)
7. [Background job queues: When to use? When not to use? Which one to use?](https://elixirforum.com/t/background-job-queues-when-to-use-when-not-to-use-which-one-to-use/20436)
8. [3 ways to schedule tasks in Elixir I have learned in 3 years working with it](https://blog.kommit.co/3-ways-to-schedule-tasks-in-elixir-i-learned-in-3-years-working-with-it-a6ca94e9e71d)
