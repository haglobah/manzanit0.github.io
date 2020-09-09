---
layout: post
title: Optimising a web crawler in Elixir
author: Javier Garcia
category: Elixir
tags: Elixir, concurrency
---

This is a really quick summary of how I optimised my personal web crawler. You
can check it out [HERE](https://github.com/Manzanit0/spidey)

## Preconditions, assumptions... things to know before starting to read

1. The intention was to create a crawler to scrape specific cooking blogs.
2. Hence, all requests are made to the same host.
3. The host with which the tests where run was
   `https://www.recetasderechupete.com`, which has around 1000 recipes.
4. This is the HTTP client module:

```elixir
defmodule Spidey.Content do
  def scan(url) when is_binary(url) do
    try do
      url
      |> get!()
      |> parse_links()
    rescue
      # Timeout, wrong url, etc.
      HTTPoison.Error -> []

      # non-html format
      CaseClauseError -> []
    end
  end

  def parse_links(html) when is_binary(html) do
    html
    |> Floki.parse_document!()
    |> Floki.find("*[href]")
    |> Floki.attribute("href")
  end

  def get!(url) when is_binary(url) do
    url
    |> HTTPoison.get!([], timeout: 15_000, recv_timeout: 15_000, follow_redirect: true)
    |> Map.get(:body)
  end
end
```

## 1st approach: Fan-out/Fan-in with Tasks

I started off with a fan-in/fan-out approach because it was the quickest way to
get started. It was a bit slow if the website happened to have a slow link in
it though.

This is broadly how it worked:

```elixir
  def scan_async([]), do: []

  def scan_async(urls) when is_list(urls) do
    urls
    |> Enum.map(fn url -> Task.async(fn -> Spidey.Content.scan(url) end) end)
    |> Enum.map(fn t -> Task.await(t, 30_000) end)
    |> List.flatten()
  end
```

Pros:

- Dead simple


Cons:

- shortest request will have to wait for the longest one


## Using pooled workers with Genservers

Since the first approach was very inneficient concurrency-wise due to the whole
thing paralising when a slow link was found, I decided to stop this by using a
pool of workers which would operate separately. Initially I tried implementing
it by myself, but ended up picking up `poolboy` for simplicity.

Pros:

- More efficient: one worker doesn't have to wait for all the other workers if
  a request takes too long.


Cons:

- Much more complex, compared to the three-liner using tasks.
- Requires either implementing a worker pool or using a 3rd party lib like `poolboy`.


## Using lazy collections

Once the concurrency solution was in place, I started optimising different
parts of the code, starting by how I was handling the crawled data.

To work with the results of each crawled site I was initially using the `Enum`
module. Since there were _a lot_ of iterations for filtering, etc. I decided to
start using lazy enumerables (the `Stream` module). This shaved off a bloody
20% of time (from ~200s before to ~160s after). Ridiculous.

These are some of the filters, although many are applied to reject static
assets, different domain urls, etc.:

```elixir
def strip_query_params(urls) do
  Stream.map(urls, fn s -> String.split(s, "?") |> List.first() end)
end

def strip_trailing_slashes(urls) do
  Stream.map(urls, fn s -> String.replace_trailing(s, "/", "") end)
end

def reject_invalid_urls(urls) do
  urls
  |> Stream.reject(&is_nil/1)
  |> Stream.reject(&(&1 == ""))
end
```

## Keeping the TCP connection alive

At this point, another important optimisation was to keep the connection alive.
All requests were being made to the same host, so it made sense that by
skipping the TCP handshakes we would shave some time. Well, this was another
big fat 10%, from ~160s to ~140s.

All as simple as telling `hackney`:

```elixir
def get!(url) when is_binary(url) do
  HTTPoison.get!(url, [],
    timeout: 15_000,
    recv_timeout: 15_000,
    follow_redirect: true,
    hackney: [pool: :default] # This is the key line.
  )
end
```

## The real deal: Domain optimisations

Another optimisation I could do is to use an ETS table as a queue instead of an
Agent/:queue, but at this stage this would be peanuts IMO, so I decided the way
to go was to optimise the actual solution to the problem. The library is a
crawler, but I developed to specifically crawl recipe websites. Taking this
into account, I started improving the filters:

- Excluding Wordpress links
- Excluding static content
- Excluding RSS feed links
- ...

And with this, I started shaving actual relevant time. And even better I learnt
a really good lesson: **Tweaking software to optimise sometimes is important,
but optimising the actual solution is what really makes it or breaks it**. Just
by excluding wordpress-specific links I was able to shave 22% of the time the
crawler took to crawl a full blog with ~1000 links, and this has nothing to do
with the software approach.

## Final thoughts

TLDR: It has been really fun to develop a half-assed worker pool in Elixir,
even thought I eventually ended up picking up a library. I've found it
incredibly satisfying to optimise an already existing solution... and I've
become VERY aware of how important it's to know what you want. Generic
solutions are great, but specific ones kick ass on the specific problems.
