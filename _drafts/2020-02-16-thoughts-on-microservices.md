---
layout: post
title: "Thoughts on Microservices"
author: Javier García
category: architecture
tags: systems, architecture, microservices
---

I've been reading Sam Newman's [Building Microservices][0], so some thoughts.

We're currently migrating to microservices in Rekki, from a big distributed
monolith.  What's a distributed monolith? - It runs in multiple instances.  We
have an ETCD lock so that one is master.


Talking via the database - using it as a crutch to migrate? It's obvious that
the shared schema is a coupling, low cohesion point though

Eventually migrate towards a HTTP/REST interface. Orchestration vs
Choreography.  Why don't we want Choreography? - RMQ is an extra piece of
complexity and it doesn't provide with enough benefits currently Would
Choreography make sense in another moment? when could it be useful?

It's never really REST, but HTTP-wannabe-REST. What are the benefits of
HATEOAS? What happens if you get it wrong? Teaching people?

## Eventual consistency

We said that using the same DB as an integration point is bad. It creates
really bad coupling. the alternative is using _retry mechanisms_, queues, or
similar systems. This ensures **eventual consistency** VS the transactional
consistency that the DB offers us.

An alternative is to rollback changes, for example by deleting the created
resources. This does increase complexity greatly though.

## Distributed transactions

**Two-phase commit**. In the first phase all _cohorts_ vote, and in the second,
the changes are made. For this there is a transaction manager in place.

This is not always a good solution because it produces locks on resources, and
it's complicated to scale. It inhibits scale AND it's not 100% foolproof.

One exaple is Java Transaction API. It's better to not handroll this stuff. The
algorithms are complex.

Futher comments from FLP:

> Whatever decision is made, all data managers must make the same decision in
> order to preserve the consistency of the database (...)

> The asynchronous commit protocols in current use all seem to have a “window
> of vulnerability”- an interval of time during the execution of the algorithm
> in which the delay or inaccessibility of a single process can cause the
> entire algorithm to wait indefinitely.

> (..) no completely asynchronous consensus protocol can tolerate even a single
> unannounced process death (...) it is impossible for one process to tell
> whether another has died (stopped entirely) or is just running very slowly.

Link [Impossibility of Distributed Consensus with One Faulty Process][3]

## Other notes on consistency

Usually, if we're talking about transactions in distributed systems, eventual
consistency is cool. If however it really is mission-critical that the stuff is
transactional, then it could be convenient to have it live in the same service
and use actual DB transactions. In these cases, creating new _concepts_ might
be a good idea, like for example an _in-progress order_, where you accumulate
all the logic before commiting it.

On another hand, I can understand that eventual consistency or distributed
transactions add an extra BIG layer of complexity that not all teams are
prepared to handle. Then, again, using crutches till we get there could be
interesting. Like everyone talking to the same DB.


## Deployment

What is the real benefit of a monorepo which contains all the services vs having each
service in it's own repo? CI becomes more complex, because you either have to deploy all
at the same time, or you have to map different pipelines to different parts of the source
tree.

On the other hand, I can see how a monorepo would make a set of teams that are not mature
enough to be conscious of the different services that exist. Also, in a green field project
it's useful as the teams discover the boundaries of the services - changes are less expensive.

Sam Newman explains how he thinks that the best approach is to go with different repositories
and each repository have it's own test suite, CI pipeline, etc. This truly makes services
more independant.

Question: Benefits of monorepos other than that? Google used monorepo, right?

## interesting reads

[Eventual Consistency Today: Limitations, Extensions, and Beyond][1]
[Reviews on the above paper][2]
[Consensus on Transaction Commit][4]



[0]: https://www.goodreads.com/book/show/22512931-building-microservices
[1]: https://queue.acm.org/detail.cfm?id=2462076
[2]: https://web.eecs.umich.edu/~mozafari/fall2018/eecs584/reviews/summaries/summary29.html
[3]: https://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf
[4]: https://arxiv.org/pdf/cs/0408036.pdf
