---
layout: post
title: "Notes on Go Concurrency"
author: Javier García
category: Go
tags: go, concurrency
---

Last year I wrote a [short post on concurrecy in Elixir][0]. I found it
absolutely fascinating to dive deeper into how the BEAM managed those green
threads, understand why the platform is fault-tolerant or even how everything
maps to lower-level primitives. Today I want to share some notes I've taken
from reading about Go.

The first thing I found interesting about Go when I started reading about its
concurrency patterns is that behind the scenes, just like Elixir, it uses
[green threads][1]. Yeah, sure, everybody talks about Goroutines, about a raw
language... but Goroutines are directly mapped to green thread. Fact. But
let's start from the beginning—let's start with the concurrency model.

## The fork-join model

Go's concurrency model is the [fork-join model][2]. It basically consists in
forking the main thread, and then recursively as many times as you want, to
eventually join or merge all those threads back into the main thread.

In Go you can of course fork and never join, but that means that when the
main thread exits, the rest of the threads will be killed without being asked
if they're finished their work or not. Give this snippet a shot:

```go
func main() {
    for i := 1; i <= 5; i++ {
        // Fork a thread
        go func(id int) {
            fmt.Printf("Hello %d\n", id)
        }(i)
    }
}
```

If you've noticed, nothing gets printed. That's because the main thread finished before
any of the forked threads where able to print anything. However, if we make sure to join
after the forks, that changes. See this other snippet:

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var wg sync.WaitGroup

    for i := 1; i <= 5; i++ {
        wg.Add(1)

        // Fork a thread
        go func(i int, wg *sync.WaitGroup) {
            defer wg.Done()
            fmt.Printf("Hello %d\n", i)
        }(i, &wg)
    }

    // We join the five threads
    wg.Wait()
}
```

But let's dive deeper. What are these threads, or goroutines as they're
called? How does the runtime create and manage them?

## Processes, threads and Goroutines

In Go, the runtime supports an M:N threading model, which is the same as as
Java and the same as Erlang. This means that it will map M green threads to N
OS threads, however, as we'll see later, the implementation is definitely
closer to that of Erlang than Java, mostly do to how threads communicate
between each other.

https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html
https://news.ycombinator.com/item?id=6977914
https://softwareengineering.stackexchange.com/questions/222642/are-go-langs-goroutine-pools-just-green-threads#

Some other resources to read, triggered from Cox's book:

https://github.com/golang/go/blob/master/src/runtime/proc.go#L16
https://github.com/golang/go/wiki/LearnConcurrency
https://en.wikipedia.org/wiki/Work_stealing
https://software.intel.com/en-us/forums/intel-cilk-plus/topic/506952
http://open-std.org/jtc1/sc22/wg21/docs/papers/2014/n3872.pdf

- Work Stealing: stealing tasks vs stealing continuation
- Go Runtime uses a pool of OS threads, and when a goroutine resumes, it pushes it to the global context.

Reread chapter 6: Goroutines and Go Runtime

----
However, while this is the main engine behind how concurrency works in Go,
there is whole lot more to it, and the reason why Go has become so popular
lately. When the Go authors started designing Go, they decided that they
wanted to have a different way to reason about concurrency. Java also
implements the fork-join model but

[0]: {% post_url 2019-09-29-elixir-concurrency.md %}
[1]: https://en.wikipedia.org/wiki/Green_threads
[2]: https://en.wikipedia.org/wiki/Fork–join_model
