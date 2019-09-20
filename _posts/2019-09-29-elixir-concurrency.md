---
layout: post
title: "Elixir: Understanding the concurrency model"
author: Javier García
category: elixir
tags: functional, elixir, concurrency
---

The first question that arises when we talk about concurrency is *what is concurrency?*

> Concurrency is the ability of different parts or units of a program, algorithm, or problem to be executed
> out-of-order or in partial order, without affecting the final outcome.
>
> ~ [*Wikipedia*](https://en.wikipedia.org/wiki/Concurrency_(computer_science))

When talking about concurrency, it's important not to confuse **concurrency** with **parallelism**. While concurrency
implies that two pieces of code are executed in different contexts, it doesn't necessarily imply that they are executed
at the same time. Parallelism does. If two fragments of code are executed in parallel, it means that the computer is
executing them at the exactly same time. Nonetheless, for parallel processing, a processor with multiple cores is needed.

## The scheduler

To understand how concurrency happens, we must first understand what **schedulers** are. A scheduler is a piece of
software, usually within the OS (Operative system), which decides which processes get executed. To decide this, it
may employ different strategies, like round-robin or first-come-first-served, among others.

In the specific case of the BEAM VM, on top of which Erlang/Elixir run, one of the underlying features it has is its
own scheduling mechanism. Every time we spin up an instance of the BEAM, it will spin up a scheduler of its own per
core in the machine's processors.

Schedulers are usually either cooperative or preemptive. **Cooperative** schedulers work by allowing threads to relinquish
control to other threads. While this has many benefits, like being very easy to implement or maximizing the resources of
the CPU, it has some pitfalls, mainly that it may cause other tasks to not meet deadlines. As you can imagine, cooperative
schedulers aren't a good fit for real-time systems.

On the other hand, **preemptive** schedulers interrupt running tasks to resume other tasks. This is usually done by time
slots or priority queues. Preemptive scheduling is usually a much better fit for real-time systems because it makes sure
that each task gets their fair share of the CPU, instead of having certains threads starve others.

In our case, Erlang uses preemptive scheduling with round robin for scheduling processes. To decide when to preempt the
the current task, it uses reductions. In a nutshell, a reduction is a counter per process which increments with each
function call, in Erlang capped at approximately 2000. This means that every 2000 function calls, the Erlang schedulers
are going to preempt the current task and have a go at the next in the queue. However, there are other ways apart from
reductions for the scheduler to stop a task. Whenever a certain process uses `receive` or any I/O operations, the
scheduler will also yield.

Since Erlang can have multiple schedulers, based on how many cores our processor has, to decrease the number of lock
conflicts in the system, it creates a run queue for each scheduler. Furthermore, To make sure that scheduling is fair
and efficient and no scheduler gets overloaded, the Ericsson team introduced the *Migration Logic* which controls and
balances queues based on statistics it collects from the system.

## Lightweight processes

Now that we've understood hoe Erlang's scheduling system works, but what about the actual processes that run our code?
How do Erlang processes look like and how do they work?

Every time we spawn a process with Elixir we are spawning what's commonly called a
[green thread](https://en.wikipedia.org/wiki/Green_threads). Green threads are processes with a very small memory
footprint that are scheduled and managed by the runtime instead of the OS. In this case, the runtime is Erlang, and the
processes are managed by the BEAM's schedulers.

Instead of spinning up native threads, Erlang creates very lightweight (~2Kb) processes which are then assigned to the
different scheduler queues. These processes will then start to run the code within, be preempted/restarted when necessary
and die once they have fulfilled their purpose.

An important property to remember of Erlang processes is that they are isolated from each other. They run concurrently,
but they don't share memory at all. Every time a process sends a message to another process that piece of information is
deep copied and passed through–this is a very important detail because it's key to avoiding resource locks.

## The actor model

With all this, Erlang uses the Actor Model for concurrency. Ultimately, the actor model is object orientation to the
upmost level–think of processes as objects, actors. An actor is the primitive unit of computation, in this case a process.
It receives messages and does something based on the messages. It's important to take into account that actors
usually come in systems, not alone. While Elixir does have the [`Task`](https://hexdocs.pm/elixir/Task.html) abstraction,
whose purpouse is to run simple tasks, most systems are designed with many processes that send messages between each other.

In order for processes to be able to send and receive messages, they have mailboxes. Every time a process is sent a
message, the message is queued up in its mailbox and isn't picked up until the process invokes `receive`. As you can
imagine, if a process doesn't *pick up its mail* regularly, it's mailbox will keep growing and potentially make the
system run out of memory. Due to this, it's important to design the behaviour of the processes taking it into account.

Like we mentioned, actors are completely isolated and never share memory, but they may have memory themselves privately.
Some examples are the [`Agent`](https://hexdocs.pm/elixir/Agent.html) or the base
[`GenServer`](https://hexdocs.pm/elixir/GenServer.html) abstractions.

Moving further with actors, what makes Erlang the resilient, fault-tolerant runtime it is, is that its processes are
able to supervise other processes and, upon certain conditions re-spawn them. **Actors can create other actors**.
This concept is key because it's the door to a self-healing system. When a supervising actor detects that one of its
children is acting in a way it's not supposed it it can kill it and re-spin it up.

## Closing

Wrapping up, I personally find that understanding the internals of how concurrency works in Elixir, and ultimately in
Erlang, is a real eye-opener when working with systems that run on top of the BEAM VM. While it's true that most of the
time we don't have to worry about what the scheduler is doing or how many reductions it takes to swap process,
knowing this opens the door to understanding what really happens under the hood every time we run
`spawn fn -> 1 + 2 end` or we open/write to a file.

While I drafted this post mainly for me, to force myself to read through the documentation and understand what happens
behind the scenes, I highly encourage you to read through some of the below resources. Specially, I have found some of
the emails shared between the Ericsson team members to be very enlightening and fun to read through.

## Resources

Some posts/threads I read through the process of drafting this blogpost:

- [How does Erlang scheduling work](https://jlouisramblings.blogspot.com/2013/01/how-erlang-does-scheduling.html)
- [Inside the Erlang VM with focus on SMP](http://erlang.org/euc/08/euc_smp.pdf)
- [Asynchronous thread pool, scheduling – email thread](http://erlang.org/pipermail/erlang-questions/2002-August/005490.html)
- [Erlang Scheduler: what does it do? – email thread](http://erlang.org/pipermail/erlang-questions/2001-April/003132.html)
- [On green threads...](https://twitter.com/joeerl/status/1010485913393254401)
- [Receiving messages in Elixir, or a few things you need to know in order to avoid performance issues](https://www.erlang-solutions.com/blog/receiving-messages-in-elixir-or-a-few-things-you-need-to-know-in-order-to-avoid-performance-issues.html)