---
layout: post
title: Bootstraping Postgres in your project
author: Javier Garcia
category: bots
tags: postgres, docker, automation
---

In the last month I've found myself having to create a bunch of greenfield
projects both for different interviews and also a couple of personal POCs I
wanted to do. The thing is, it made me realise that I never really put much
time into figuring out the most comfortable way of setting up a Postgres
database for a new project.

Before, for most of my projects, any time I needed to connect to a database I
would either connect to my local running service, or directly to the instance
in Heroku or AWS. The problem is when a new person is going to pick up the
project, it requires a lot of manual setup for them. This made me want to
automate this 100%.  And not just for these projects I was working on, but for
every other new future project in my life. I wanted to just copy-paste a couple
of files into my project and know that it would "just work".

## Oh Docker, where art thou?

The first thing I realised is that the best option was docker. Any other option
would involve messing with the local service or a remote one, thus throwing out
of the window the automation.

What's the easiest way to get a dockerised postgres running? This:

```sh
docker pull postgres
mkdir -p $HOME/docker/volumes/postgres
docker run --rm   --name pg-docker -e POSTGRES_PASSWORD=docker -d -p 5432:5432 -v $HOME/docker/volumes/postgres:/var/lib/postgresql/data postgres
```

But this is nowhere near enough. First, this is three comands, second it
doesn't initialise the database for my application and third stuff is hardcoded
there all over the place and I want to make it really easy to just copy paste
some stuff to a new project. We don't have to go too far for this, straight
from the [Dockerhub][0] documentation we can find an initialisation script and
we can just push this to a Makefile to start making it more accessible.

The init file (`/me/my-project/scripts/db/init-db.sh`):

```bash
#!/bin/bash
set -e

psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
    CREATE USER docker WITH PASSWORD '123';
    CREATE DATABASE memento_db;
    GRANT ALL PRIVILEGES ON DATABASE memento_db TO docker;
EOSQL
```

And the Makefile:

```Makefile
POSTGRES_CONTAINER_NAME := memento-pg
POSTGRES_PASSWORD := password
PSQL_USER := docker
PSQL_PASSWORD := 123
DATABASE_NAME := memento_db

db-start:
	docker pull postgres && \
	docker run --rm --name $(POSTGRES_CONTAINER_NAME) \
		-p 5432:5432 \
		-e POSTGRES_PASSWORD=$(POSTGRES_PASSWORD) \
		-v $(PROJECT_DIR)/scripts/db/:/docker-entrypoint-initdb.d \
		-d postgres
```

Now we're talking! `make db-start` and we got our database running, already
initialised :)

## What about migrations?

The next thing on my mind was migrations. Most ecosystems deal with this in
their own way. In Elixir, you can use Ecto's baked in mechanisms, .NET has
Entity Framework's mechanics, and so on. However, I wanted something which was
language-agnostic, something that I could use everywhere if I wished so. That's
how I decided to use Flyway and raw SQL.

I won't get into too much detail because there already is an official
documentation for that, but in a nutshell, all I had to do was create a
`migrations/` directory under the project root folder, create a new `*.sql`
file for each new migration and run Flyway.

Just like for the database, since I don't want to burden myself with having to
setup Flyway in the computer I'm working with every time, use docker. This
would be the make rule:

```Makefile
PROJECT_DIR := $(shell dirname $(realpath $(firstword $(MAKEFILE_LIST))))
PSQL_USER := docker
PSQL_PASSWORD := 123

db-migrate:
	docker run --rm --network="host" -v $(PROJECT_DIR)/migrations:/flyway/sql dhoer/flyway:alpine \
		-url="jdbc:postgresql://localhost:5432/$(DATABASE_NAME)" \
		-user=$(PSQL_USER) \
		-password=$(PSQL_PASSWORD) \
		-schemas=public \
		-connectRetries=3 migrate
```

The rule is pretty self-explanatory, however I'll note two things: first is
that we have to enable the container to access the host's network. At the end
of the day, we want two containers to talk to each other. To do this we expose
the database container's 5432 port and we enable the flyway container to access
the network.

The second thing is that we're using the user which we configured in the
initialisation script, not the superuser. Just for security, nothing special
about it.

## Putting the Makefile together

The last thing I would add to this is a way to access the database so we
can directly talk to it during our development workflow. Many people use GUIs, I'm
personally a fan of the terminal, so I'll suggest `pgcli`.

```Makefile
pgcli:
	docker run -it --rm --network="host" dencold/pgcli postgresql://$(PSQL_USER):$(PSQL_PASSWORD)@localhost:5432/$(DATABASE_NAME)
```

This would be the full Makefile with a couple of extra additions:

```Makefile
POSTGRES_CONTAINER_NAME := memento-pg
POSTGRES_PASSWORD := password
PSQL_USER := docker
PSQL_PASSWORD := 123
DATABASE_NAME := memento_db

PROJECT_DIR := $(shell dirname $(realpath $(firstword $(MAKEFILE_LIST))))

pgcli:
	docker run -it --rm --network="host" dencold/pgcli postgresql://$(PSQL_USER):$(PSQL_PASSWORD)@localhost:5432/$(DATABASE_NAME)

db: db-teardown db-start db-migrate

db-start:
	docker pull postgres && \
	docker run --rm --name $(POSTGRES_CONTAINER_NAME) \
		-p 5432:5432 \
		-e POSTGRES_PASSWORD=$(POSTGRES_PASSWORD) \
		-v $(PROJECT_DIR)/scripts/db/:/docker-entrypoint-initdb.d \
		-d postgres

db-teardown:
	docker stop $(POSTGRES_CONTAINER_NAME) || true && \
	docker rm $(POSTGRES_CONTAINER_NAME) || true

db-migrate:
	docker run --rm --network="host" -v $(PROJECT_DIR)/migrations:/flyway/sql dhoer/flyway:alpine \
		-url="jdbc:postgresql://localhost:5432/$(DATABASE_NAME)" \
		-user=$(PSQL_USER) \
		-password=$(PSQL_PASSWORD) \
		-schemas=public \
		-connectRetries=3 migrate
```

I was so happy when I ended up with that Makefile... took me barely a couple of
hours of playing around with the different parts, but at the end, it's
something you can copy/paste in any project and it _just works (TM)_. Now, next
time I start a new project, I can simply copy it along with the
`scripts/db/init-db.sh` script and the `migrations` directory and start hacking
away.

## Alternative: docker-compose

In the midst of playing, I also gave a go at `docker-compose` and came to the following:

```yaml
version: '3'

services:
  postgresql:
    image: postgres:9.5-alpine
    restart: always
    volumes:
      - ./scripts/db:/docker-entrypoint-initdb.d/
    environment:
      - POSTGRES_USER=root
      - POSTGRES_PASSWORD=password
    ports:
      - 5432:5432

  flyway:
    image: flyway/flyway
    command: -url=jdbc:postgresql://postgresql:5432/memento_db -schemas=public -user=docker -password=123 -connectRetries=60 migrate
    restart: on-failure
    volumes:
      - ./migrations:/flyway/sql
    depends_on:
      - postgresql
```

And it works just as fine. If you're application is also containerised, you can
even chuck it in there and have the full thing spin up with a single command. I
ended up prefering the Makefile because that way I can just run the application
on my computer, no containers involved, and I prefer it, but they're not
exclusive.  You could potentially have both, and use them at will. Add a couple
extra rules to your Makefile and you're good to go:

```Makefile
up:
	docker-compose up --build

teardown:
	docker-compose -f docker-compose.yml down --volumes
```

## Final thoughts

With all this, all I can say is databases should no longer be a pain when
setting up a new project. They're not for me any more, just another minor thing
like importing `jest` or copying some snippet from StackOverflow. I hope this
helps you as much as its helped and will help me!

[0]: https://hub.docker.com/_/postgres
