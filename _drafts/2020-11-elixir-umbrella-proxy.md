---
layout: post
title: Routing in Elixir Umbrellas
author: Javier Garcia
category: elixir
tags: elixir, umbrella
---

While I'm not a huge fan of umbrella applications in Elixir, they do exist for a purpouse — providing
developers with a means to have multiple applications managed under a same repository with common tooling. In
my experience, there's two basic flavours of umbrella applications: those in which each application
has its own deployment lifecycle, and those in which applications are simply OTP applications or grouped
modules and the whole umbrella itself acts as a monolith. Today I want to talk about the latter.

In one of the projects I'm currenty working on, we have a monolithic umbrella with multiple applications,
mostly to seggregate concerns, and we recently decided it was time to add a backoffice. Initially we had
two applications; one with the core business logic and another with an absinthe API, but creating a Phoenix
backoffice forced us to take one of two paths: either build it within the API application and sharing router,
or building it as a separate application with it's own thing going on. The challenge of the latter was that
while locally all is good because you can have each HTTP server running on a different port, that's not viable
in a live server because HTTPS is served through port 443, and HTTP through port 80. You don't get to pick
and choose.

## Running multiple apps under the same server

Here's when the talk ["building an umbrella project"][0] at ElixirConf USA 2016 came in handy. The solution
they proposed was basically creating a single application under the umbrella whose only purpouse is to
direct the requests to the right router either based on host or request path. All the other apps would simply
expose the router and the proxy would multiplex.

So, taking this example as a base, I decided to take off and build on top of it. Being honest, the only extra
thing I added on top of it was to use a Phoenix endpoint instead of plug, simply because Phoenix takes care
of some extra niceties.

This is the new project structure with the new proxy application:

```
my_umbrella/
  apps/
    api/
    backoffice/
    core/
    proxy/
      lib/
        proxy/
          endpoint.ex
        proxy.ex
      mix.exs
```

And this is how both the proxy endpoint and the application look like:

```elixir
defp deps do
  [
    {:api, in_umbrella: true},
    {:backoffice, in_umbrella: true},
    {:phoenix, "~> 1.5"},
    {:plug_cowboy, "~> 2.0"}
  ]
end
```

```elixir
defmodule Proxy.Endpoint do
  use Phoenix.Endpoint, otp_app: :proxy
  require Logger

  def init(opts), do: opts

  def call(%Plug.Conn{} = conn, _) do
    if conn.request_path =~ ~r{/admin} do
      Backoffice.Endpoint.call(conn, [])
    else
      API.Endpoint.call(conn, [])
    end
  end
end
```

```elixir
defmodule Proxy do
  use Application

  def start(_type, _args) do
    children = [
      Proxy.Endpoint
    ]

    opts = [strategy: :one_for_one, name: Proxy.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

## Configuring the project

TODO: how to keep the live reloading in dev environment

[0]: https://github.com/wojtekmach/acme_bank
