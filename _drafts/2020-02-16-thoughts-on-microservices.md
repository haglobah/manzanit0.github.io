---
layout: post
title: "Thoughts on Microservices"
author: Javier García
category: architecture
tags: systems, architecture, microservices
---

I've been reading Sam Newman's [Building Microservices][0], so some thoughts.

We're currently migrating to microservices in Rekki, from a big distributed monolith.
What's a distributed monolith? - It runs in multiple instances.
We have an ETCD lock so that one is master.


Talking via the database - using it as a crutch to migrate? It's obvious that the shared schema
is a coupling, low cohesion point though

Eventually migrate towards a HTTP/REST interface. Orchestration vs Choreography.
Why don't we want Choreography? - RMQ is an extra piece of complexity and it doesn't provide with enough
benefits currently
Would Choreography make sense in another moment? when could it be useful?

It's never really REST, but HTTP-wannabe-REST. What are the benefits of HATEOAS? What happens if you get it wrong? Teaching people?

[0]: https://www.goodreads.com/book/show/22512931-building-microservices
