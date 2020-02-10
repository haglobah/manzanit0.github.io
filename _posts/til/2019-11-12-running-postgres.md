---
layout: post
title: "Cheatsheet: starting and stopping Postgres"
author: Javier Garcia
category: Postgres
tags: postgres, ubuntu, terminal
---

I always find it complicated to remember how to start/stop postgres since I have different setups
in different places, so I figured it would be convenient to centralize that knowledge so I can come
back to it. Below is the cheatsheet :)

_Disclaimer: This has been run in Ubuntu 18.04._

## Running Postgres locally

To start the postgres server:

```
sudo /etc/init.d/postgresql start
```

To stop the server:

```
sudo /etc/init.d/postgresql stop
```

## Running Postgres in Docker

To download and run docker as a Docker container:

```
docker pull postgres
mkdir -p $HOME/docker/volumes/postgres
docker run --rm   --name pg-docker -e POSTGRES_PASSWORD=docker -d -p 5432:5432 -v $HOME/docker/volumes/postgres:/var/lib/postgresql/data postgres
```

To stop it, simply kill the container. Some useful commands:

```
docker container ls
docker stop <container>
docker kill <container>
```

## And to connect...

```
psql -h localhost -U postgres -d postgres
```