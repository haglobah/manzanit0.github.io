---
layout: post
title: "Elixir: Monorepos"
author: Javier Garcia
category: elixir
tags: elixir, monorepos
---

One of the most recurring conversations of 2021 with different work colleagues
has been about monorepos; if they're good or bad, if the promote easy to read
PRs or if they empower unhealthy code practices.

Roughly I'd define a monorepo as a single version controlled repository where
the code of multiple applications live. Normally, small systems get away with
leveraging a monolith, a single application that handles all the needs of the
system, be it backend, frontend or both, thus simplifying greatly the
infrastructure. However, as systems become more complex it is quite common to
slowly support the initial application with other applications. This is where
the concept of a monorepo comes into play: we can store the source code of new
applications we develop in new repositories, or keep them in the same
repository.

After looking back into all those discussions, I figured it could be
interesting to bring some of it into a post: why I find monorepos are a good
choice in some scenarios, how to set up a sane monorepo so both the local and
the cloud stories make sense, and ultimately how this would all look like in
the specific case of Elixir. Most of what I intend to write about could apply
to any language, but I decided to pick Elixir as the showcasing language
because I believe that it's an ecosystem that could greatly benefit from more
simple service-based architectures with monorepos considering the default are
monoliths. Not that one is better than the other, but sometimes it may feel
that most literature writes about writing [majestic monoliths][13] leaving
microservice architectures as a thing for other ecosystems such as Go or Java.

## Creating a sane monorepo

Obviously there is an infinite amount of ways we can decide to organise our
code in a repository, specially as we begin to push more applications to the
same repository. However, what makes a structure good? How can we keep
everything in the same place without loosing hope in humanity as the system
grows?

The first thing to take into consideration is that for a monorepo to be useful
it must enhance discoverability, as it's one of it's key strengths. Any new
developer to join the team must be able to at least understand the outline and
be able to infer where things sit. In this regard, I usually recommend
segregating shared code from application code and having a clear separation of
applications. Having a directory called `apps` which contains a set of
subdirectories each for each application and another `libs` for shared libs is
a good starting point. I always recommend folks to also **always** add READMEs
to each application as well as each parent directory to help make the
repository discoverable.

In the specific case of Elixir, there's a type of application which can be
confused with a monorepo; those are [umbrella applications][16]. While umbrella
projects initially seem like a monorepo because they seggregate applications,
it's slightly different. An umbrella project is used to build applications that
run together in a single repository, but as such those applications are not
fully decoupled: they all share the same configuration and the same
dependencies. And this is something to keep in mind. Umbrella projects do
provide some conveniences to compile, test and run all the application at once,
but ultimately they're more like a project in themselves. If what you need is a
set of independent applications stored in the same repository, then you're not
looking at an umbrella.

On top of a good organisation, another key thing to consider when adopting a
monorepo is that tooling can easily become a challenge - it can be that our
monorepo holds the code of applications written in different languages, or
simply that as the amount of code grows the tooling to manage it needs to
become more performant. This is something [Google has stumbled against
themselves][15].

Regarding tooling, at least initially,  I believe that attempting to generalise
as much as possible allows for faster iteration, and optimisation can be
tackled as issues arise. Generic tooling means knowledge from working with one
application to another is transferable, on top of that it means that adding a
new application to the monorepo becomes trivial, because the same tooling can
be leveraged. For this, docker is a great option. By wrapping our applications
in containers, we can start to interact with them (booting them up, tailing
logs, deploying them, etc.) in a generic way without having to individually
automate a flow for each one of them. I'll touch more on this topic later, but
ultimately we want to make tooling simpler when possible as we work in a
monorepo.

## Why does a monorepo make sense, or not?

A fairly common question is *what makes a monorepo a good candidate for the
problem I'm trying to solve?*. Most readers may be reading this with the
organisation they work in in mind, leveraging the monorepo to centralise the
organisation's code, but a monorepo may be used for smaller focused problems
too.

I believe monorepos can be good solutions to certain problems. They are, for
example a great solution to grouping a set of distributed services with high
cohesion, or for quickly scaffolding a new project whose boundaries aren't too
clear. By keeping all the code in the same repository I can easily refactor
HTTP interfaces and quickly discover consumers because I can search in the same
repository, or I can make cross-cutting changes through multiple applications,
such as the API and the UI that consumes it in the same pull request.
Ultimately, the main two advantages of monorepos are discoverability of
projects and refactorability of code.

One of the advantages of monoliths has always been that refactoring the code to
better decouple code is an easy task, however, by leveraging a monorepo this
also becomes easy in microservice architectures. It is undeniable that it comes
with other costs, such as having too coordinate with the owners of other
projects in the repository before merging our changes, but the refactor in
itself is easy. Changing a library without knowing the code that consumes it
can be a daring task.

On the other hand, if you find yourself struggling with versioning, such as
needing different versions of a package for different applications, then
keeping that package along with the applications might not be the best option.
We'll explore this later.

At the end of the day, **a monorepo should be about making our lives easier,
not harder**. In the context of an organisation, I personally find it useful to
manage the services a single team uses. It empowers that one team to make
cross-cutting changes across all the services they manage without having to
jump repositories and being able to see the whole impact in a birds-eye view.

## Case Study: Small Elixir Services

To start looking at the lights of a monorepo, we're going to use Elixir. I
decided on Elixir for two reasons: the community mostly focuses on monoliths,
which don't really end up leveraging monorepos all that often, and secondly
because I'd like to showcase how easy Elixir makes it build small simple
services that we can scale as much as we need **without leveraging distributed
erlang**; just standard immutable containers.

For the sake of the blog post, let's assume that we're trying to build a system
that exposes an API to log users in. We need to expose an endpoint such as
`POST /log_in` which takes the user and password and simply returns us a token
to do any further actions we want to. On top of this, we want to also send the
user an email every time he logs in for security reasons. However, we want to
develop the email processing in a different service because we know that it
will receive much more load than the initial service; it could potentially be
leveraged by the wider system for sending every email.

This means that we should end up having something like this:

```
+-------------+                +--------------+              +---------------+  
|             |  POST /log_in  |              |  PUT /email  |               |  
|    User     |--------------->| Auth Service |------------->| Email Service |  
|             |                |              |              |               |  
+-------------+                +--------------+              +---------------+  
```

Lastly, not that we'll explicitly need it for this case, but we'll attempt to
wrap the email service's client in a package of it's own so that as we move
forward, every other service we build doesn't need to write their own client.
With this we'll showcase how to share code.

While we're not going to really dive into the code itself because I intend to
focus on more on the cross-cutting concerns, you may want to try the
bootstraping, as well as the code sharing techniques I'll share. If that's the
case, to keep up with the post I suggest you simple create a directory with the
following structure:

```
 apps/
    auth_service/
    email_service/
 libs/
```

You can get started with the following commands:

```sh
$ mkdir apps/
$ cd apps
$ mix phx.new auth_service --no-assets --no-html --no-gettext --database postgres
$ mix phx.new email_service --no-assets --no-html --no-gettext --database postgres
```

## Running distributed components

Like we mentioned earlier, one of the techniques that makes working with
monorepos easier is leveraging containers for our applications. This empowers
us with a uniform way to build and run applications regardless of the tecnology
beneath, be it Elixir, Python or C++. In this regard, we can decide for our
development story to be different than our deployment one, that is, we could
potentially leverage Docker for local development and then use Heroku
buildpacks for deployment, we could use containers for both, or we could go
bare metal and shell scripts for local and something else for deployment.
That's up to the team. However, I encourage leveraging container technologies
due to how much it simplifies the application lifecycle.

In Elixir, the best way to containerise applications is leveraging
[releases][1]. Ultimately, a release is a self-contained directory which
contains the application code, its dependencies and the whole Erlang VM and
runtime. An assembled release can be deployed as-is to any target as long as it
runs the same operating system as the assembling machine.

This doesn't mean that we need to use releases to containerise an application.
We could perfectly run a container with Elixir installed, copy the repository
in it and run `mix start`, but do keep in mind that `mix` has been designed as
a build tool, and as such it has some drawbacks. Amongst them, the main
difference is that releases eagerly load the whole application and its
dependencies while `mix` does so lazily, meaning that there could be a big
latency spike the first time requests start. This said, even though I'd argue
that [releases are the way to go almost always][17], if it's something that
makes sense for your use case, running the application with `mix` is also an
option, and it can be achieved through containers just as well. Containers are
a wrapper around however you run your application, just that.

To take advantage of releases, the setup is minimal, Elixir makes it very easy.
If we're leveraging [Phoenix][2] we can run `mix phx.gen.release` to scaffold
some useful modules followed by `mix release` to build the actual release, if
we're not, then it's as simple as just running `mix release`. Do note that you
don't need Phoenix to create a release though. I've gotten away a few times
with running a barebones [`cowboy`][3] server with a couple of endpoints
without any fluff. ElixirSchool has [a nice resource][8] on this.

If you're running an umbrella project, it does require some extra ceremony to
create the releases, such as adding some extra configuration in the `mix.exs`.

Ultimately, the flow is: we code our application, we bundle it into a release,
and then ship it in a container. The specific how can vary depending on the
usecase. Personally, I like copying the source code into the container image
and then compiling and building the release in it. This has the added benefit
that anybody can run the application in the computer without having Elixir
installed. In the context of a monorepo which may be used by many teams, it
empowers other teams to bootstrap our application without having to learn or
understand any part of the development toolkit.

This said, how does that `Dockerfile` look like then? A fairly elaborated
example can be found in the [Phoenix documentation][4]. However, assuming we
don't have any frontend assets and we want to go minimalistic, it can be as
simple as this:

```Dockerfile
FROM elixir:1.12.3-alpine

RUN apk add --no-cache build-base npm git

WORKDIR /app

RUN mix local.hex --force && \
    mix local.rebar --force

ENV MIX_ENV=prod

COPY mix.exs mix.lock ./
COPY config config
RUN mix do deps.get, deps.compile

COPY lib lib
COPY priv priv

RUN mix do compile, release

RUN chown -R nobody:nobody /app/_build/prod/rel/email_service
USER nobody:nobody
WORKDIR /app/_build/prod/rel/email_service

CMD bin/email_service start
```

How your `Dockerfile` will look and the process to creating a release will
greatly depend on your application, any external dependencies it may leverage
and how you want to set it up. For example, it's a common practice to run the
migrations before starting the new server, so some folks might leverage the
`Release.ex` module created by `mix phx.gen.release` and make the Docker `CMD`
such as:

```Dockerfile
CMD bin/email_service eval "EmailService.Release.setup()" && bin/email_service start
```

Once you've dockerised your applications, the problem that usually arises is
*how do I run them together without too much friction?*. Unless you have very
complex needs, I usually just go with [`docker compose`][5]. I won't go into
the details, but an example `docker-compose.yml` which also runs Postgres could
be:

```yaml
version: '3'
services:
  db:
    image: postgres:13-alpine
    healthcheck:
      test: ["CMD", "pg_isready", "-q", "-d", "postgres", "-U", "root"]
      timeout: 45s
      interval: 10s
      retries: 10
    restart: always
    environment:
      - POSTGRES_USER=root
      - POSTGRES_PASSWORD=password
    ports:
      - "5431:5432"
    volumes:
      # -- The script can be something as simple as:
      # CREATE USER docker WITH PASSWORD '123';
      # CREATE DATABASE email_service_db OWNER docker;
      # CREATE DATABASE auth_service_db OWNER docker;
      - ./docker/db:/docker-entrypoint-initdb.d/

  email_service:
    depends_on:
      - db
    build:
      context: apps/email_service
    environment:
      SECRET_KEY_BASE: "S3croj/0ZcfwSMYKKhzNrRnCFcdSNVrrO+Pt1nnRYdtiOWkQj08hUVwMdRkk8BL+"
      DATABASE_URL: "postgres://docker:123@db/email_service_db"
      PHX_SERVER: true

    ports:
      - "4001:4000"

  auth_service:
    depends_on:
      - db
      - email_service
    build:
      context: apps/auth_service
    environment:
      EMAIL_SERVICE_URL: "http://email_service:4000"
      SECRET_KEY_BASE: "Q2MF2Q+lE9uNqRmylF6LMV4u7sYgpaA/FinEzUTKQo5ea+Uci0OZwnKb+EcatGp3"
      DATABASE_URL: "postgres://docker:123@db/auth_service_db"
      PHX_SERVER: true
    ports:
      - "4002:4000"
```

With this setup, if we want to have the whole system up and running, it's as
easy as running `docker compose up`, and then `docker compose down` to tear it
down. In terms of DX, if we want to work on a specific component, we can stop
it with `docker compose stop [SERVICES...]` and then run it on bare metal with
[`mix phx.server`][6] or [`mix start --no-halt`][7], depending on the project.
This allows for a faster feedback loop upon recompiling.

Keep in mind that `docker compose` is a toolkit for running containers locally.
Orchestrating distributed systems at scale is a whole different story and for
that we would need to start talking about Docker Swarm or Kubernetes (k8s). We
won't be covering that in this post though.

## Sharing code between components

At this point we have two dockerised services and a way to boot them up in
tandem. As we move towards coding a shared library, how does that look? Like we
mentioned earlier, to help discoverability in a monorepo, keeping shared code
in their own structure, treating them as separate libraries is a big advantage.
I usually advocate in favour of having a `libs` or `pkg` directory where
we can push all those to.

Getting started, there are a few quirks both on the Elixir and the docker side,
but let's start with Elixir. Creating a library in Elixir is quite straight
forward. Considering the structure we outlined initially, we can create any
shared code under the directory `libs`:

```sh
mkdir libs
mix new email_service_client
```

Having created our library and assuming we're using something like [`Tesla`][9]
to write our HTTP client, then code may look something like this:

```elixir
defmodule EmailServiceClient.Client do
  def send_email(attrs) do
    {id, attrs} =
      # Note: simple hack to create new ids; it doesn't work for a production system though.
      Map.pop_lazy(attrs, :id, fn -> System.unique_integer([:monotonic, :positive]) end)

    Tesla.put(client(), "/api/emails/#{id}", attrs)
  end

  defp client do
    config = Application.get_env(:auth_service, AuthService.EmailClient)

    middleware = [
      {Tesla.Middleware.BaseUrl, config[:base_url]},
      Tesla.Middleware.Telemetry,
      {Tesla.Middleware.Headers, [{"content-type", "application/json"}]},
      Tesla.Middleware.JSON,
      {Tesla.Middleware.Timeout, timeout: 2_000},
      Tesla.Middleware.Logger
    ]

    Tesla.client(middleware)
  end
end
```

However, how can we leverage this new library we just created in the monorepo
from our different services? Well, Elixir provides for [two different ways][14]
of importing a library: either though an existing repository or through a
relative path. In our case, let's explore the relative path option.

If we've created the repository with a structure as mentioned initially, that is:

```
apps/
  auth_service/
  email_service/
libs/
  email_service_client/

...
```

Then we can import our new package by adding to the our `mix.exs`:

```elixir
{:email_service_client, path: "../libs/email_service_client"}
```

And that's it! If we now compile our service, notice how it's pulling in the
shared library and compiling it too.

Regarding docker though, we'll need to make some amends both to our
`Dockerfile` and to `docker-compose.yml`, mostly due to the concept of the
[context][10]. When `docker compose` starts building the image, it needs access
to the parent directory in order to access the shared lib. This cannot be done
in docker other that providing a different context; the parent directory:

```yaml
services:
  auth_service:
    build:
      context: .
      dockerfile: apps/auth_service/Dockerfile
```

However, since the context is now different, the `Dockerfile` is invalid we
need to amend it as if we're working though the parent directory:

```Dockerfile
FROM elixir:1.12.3-alpine

RUN apk add --no-cache build-base npm git

WORKDIR /app

RUN mix local.hex --force && \
    mix local.rebar --force

ENV MIX_ENV=prod
ENV SHARED_LIBS_DIR=shared

COPY /apps/auth_service/mix.exs /apps/auth_service/mix.lock ./
COPY /apps/auth_service/config config

# Not very efficient since we're copying all of them.
COPY /libs shared

RUN mix do deps.get, deps.compile

COPY /apps/auth_service/lib lib
COPY /apps/auth_service/priv priv

RUN mix do deps.get, deps.compile

COPY /apps/auth_service/lib lib
COPY /apps/auth_service/priv priv

RUN mix do compile, release

RUN chown -R nobody:nobody /app/_build/prod/rel/auth_service
USER nobody:nobody
WORKDIR /app/_build/prod/rel/auth_service

CMD bin/auth_service start
```

And at this stage, everything should run smoothly.

From a "clean code" perspective, you may notice though that we have a
Dockerfile within a child directory with knowledge of the parent directory. Not
that there is anything wrong per-se with it, but as the monorepo grows, you
might want to consider extracting the docker layer into it's own root folder.
It's a way of isolating that knowledge in it's own corner: it could be argued
that the containerisation of applications doesn't have anything to do with the
application in itself but more with the ops of the wider system. Regardless,
that's up to you.

One last remark to make on this point is to consider versioning. It's not too
uncommon that as a monorepo grows shared libraries evolve but not every service
evolves with them, which means that you could end up being in a position where
a service is running a version of a library that was long lost in git history.
There's not really any way around this if we reference shared code though path.
In this case we should probably look into either extracting the library into an
external repository, so we can use branches for versions, or publish the
package into a private repository, which [Hex allows][11]. This does highlight
the dichotomy between monorepos and polyrepos. They both have strengths and
weaknesses.

## Setting up CI

As we finish our services and start preparing for deployment, a good thing we
can add to our projects is CI (Continous Integration). With CI you can go as
far as your like, from checking only that the project compiles all the way to
running the tests, checking code coverage and even looking for code smells.
However, in monorepos this is one of the sensitive topics. What do we run CI
on? Do we want to lint the whole repository or just our one project? Should the
auth service be allowed to deploy if the email_service doesn't compile?

This is a can of worms and a whole other topic to discuss, but in a nutshell,
I'm usually in favour of decentrilising the ops in a monorepo. Ultimately, the
whole purpouse of a microservices architecture is to scale the organisation,
and by holding every service accountable for every other service's status we're
cancelling this out. This means that we can benefit much more from linting both
the auth and the emails services separately. Github Actions offers us an easy
way to do this: we can run worflows per directories.

Like we said at the beginning, we can go as far as we want, but I'll share a
simple CI job I use quite often: it compiles, caches the build for speed and
runs the test. Most of the times, thanks to how well Elixir plays with tooling,
you can get away copy-pasting these:


```yaml
name: "auth"

on:
  push:
    # run the workflow only on changes
    # to the auth route and auth workflow
    paths:
      - "apps/auth_service/**"
      - ".github/workflows/auth.yaml"

jobs:
  test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        elixir: [1.12.3]
        otp: [23.2.7]

    services:
      db:
        image: postgres:12-alpine
        ports: ["5432:5432"]
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: auth_service_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v2

      - name: Setup Elixir
        uses: erlef/setup-beam@v1
        with:
          elixir-version: ${{ '{{' }} matrix.elixir }}
          otp-version: ${{ '{{' }} matrix.otp }}

      - name: Retrieve Mix Dependencies Cache
        uses: actions/cache@v1
        id: mix-cache
        with:
          path: apps/auth_service/deps
          key: ${{ '{{'}} runner.os }}-${{ '{{' }} matrix.otp }}-${{ '{{' }} matrix.elixir }}-mix-${{ '{{ '}} hashFiles(format('{0}{1}', github.workspace, '/apps/auth_service/mix.lock')) }}

      - name: Install Mix Dependencies
        if: steps.mix-cache.outputs.cache-hit != 'true'
        run: |
          cd apps/auth_service
          mix local.rebar --force
          mix local.hex --force
          mix deps.get

      - name: Check Formatting
        run: |
          cd apps/auth_service
          mix format --check-formatted

      - name: Compile project without warnings
        run: |
          cd apps/auth_service
          mix compile --warnings-as-errors

      - name: Run tests
        run: |
          cd apps/auth_service
          mix test
```

## Deploying the system

Coming to an end, the last step I wanted to add some remarks about is
deployment, although nothing too specific. The Phoenix docs have a [multiple
guides][12] on this topic.

When deciding how to deploy our different applications within the monorepo,
something to take into consideration is that the same issues as when running
the CI come up: deploying everything in tandem or deploying each component
individually, deploying manually or deploying through CD (Continous
Deployment). Just like with CI, I believe that it works best to always deploy
components individually, otherwise the couplement is terrible. If we deploy
everything together, we could be deploying a broken service that was
rolled-back the day before and a fix has not been put in place yet. And we
didn't know. In this scenario, we end up having the worst of a distributed
architecture and a monolithical one.

Nonetheless, consider it. At the end of the day, it's context dependant. If
you're running a monorepo but you're in charge of all the services, then it
might just pay off for you. What I usually like is for deploys to be short and
snappy, and to be manual, so I can launch them from my terminal, but that's me.

## Wrapping up

All this said, there is not silver bullet when it comes topics like monorepos.
There aren't solutions that work all the time as well as there aren't ones that
never work. I personally have found that working with small distributed Elixir
services is a joy; not every project warrants a monolith, and using a monorepo
does allow me to group those services by domain quite easily. Elixir provides a
nice way to import shared packages, and fantastic tooling to easily
containerise applications.


[1]: https://elixir-lang.org/getting-started/mix-otp/config-and-releases.html#releases
[2]: https://www.phoenixframework.org/
[3]: https://hexdocs.pm/plug_cowboy/Plug.Cowboy.html
[4]: https://hexdocs.pm/phoenix/releases.html
[5]: https://docs.docker.com/compose/
[6]: https://hexdocs.pm/phoenix/Mix.Tasks.Phx.Server.html
[7]: https://hexdocs.pm/mix/1.12/Mix.Tasks.Run.html
[8]: https://elixirschool.com/en/lessons/misc/plug
[9]: https://github.com/teamon/tesla
[10]: https://docs.docker.com/engine/reference/commandline/build/
[11]: https://hex.pm/docs/private
[12]: https://hexdocs.pm/phoenix/deployment.html
[13]: https://m.signalvnoise.com/the-majestic-monolith/
[14]: https://hexdocs.pm/mix/1.12.3/Mix.Tasks.Deps.html
[15]: https://dl.acm.org/doi/10.1145/2854146
[16]: https://elixir-lang.org/getting-started/mix-otp/dependencies-and-umbrella-projects.html#umbrella-projects
[17]: https://elixirforum.com/t/always-use-releases/4573
