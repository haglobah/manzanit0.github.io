---
layout: post
title: Thoughts on Microservices
author: Javier García
category: architecture
tags: systems, architecture, microservices
---

I recently finished Sam Newman's [Building Microservices][0], so I figured it
would be a good a moment as any other to share some thoughts on the topic,
patterns I've seen and overall experiences.

## Unconventional approaches

Back when I worked at REKKI, I had the chance to work on an unconventional yet
truly interesting migration from a big distributed monolith to a services
architecture.

In regards to the monolith, just think of a big fat Elixir/Phoenix app which
runs in multiple instances. We leveraged an ETCD lock to make one master, and
some operations are handled by the master others are load balanced among the
different servers. There was some extra complexity in the system, since there
was a RMQ server around there, the monolith was actually a very Elixir-specific
type of monolith which allowed us to run different parts of it separately, and a
few other details, but it's not all that relevant for what I wanted to share.

Regarding the migration, it was unconventional mostly for one reason: we decided
to start developing small Go microservices... but keep using the same database -
a beautiful Postgres instance. Ocasionally a service would make a request to
another, but since the state was shared... most of the times it was as easy as
just shooting a query to the DB. Ultimately, services where talking to each
other through the database.

From this approach, certain patterns emerged: cronjobs which would run every
minute and query a certain table to run certain commands, services which would
lock tables to make sure that their operations were consistent, domain models
which were shared across services... and like everything, some were good, others
had more drawbacks.

The interesting part is that one of the thing Sam Newman states in his book is
that sharing the database is a no-no, or at least highly discouraged, mainly
because it couples services together, hence defeating the purpouse. However,
after working with said architecture for around 6 months... I'd say it just
depends. I actually liked it.

While it is absolutely true that by having 20 services share the same database
as well as the same repository, packages, etc does couple them greatly, it is
also true that it empowers developers with unprecedented iteration speed. And
context here is key: we were a very early stage startup which had not found
product-market fit, the team was very heterogenous in terms of skills, and
changes had to be meaninful to the users. A week of refactors without
user-impact was very costly.

Suddenly we found ourselves being able to simply try out new ideas in an
encapsulated service, deploy it within 2 minutes and if after a few weeks we
realised that it wasn't what we needed, we just burnt it. This meant new
frontend apps, new HTTP services, new jobs, anything. The data layer was the
same, so no need to "start from scratch" on that end, it was just a matter of
figuring out the meaningful business logic and, above all, new developers didn't
have to learn a massive new codebase. They just had to understand the data
model in order to start coding their services, no context required.

Nonetheless, in the short span of time I worked with this new architecture I did
have the chance to see some of its shortcomings: jobs which queried the database
periodically emerged where it could have been a service being called, services
started to need to lock certain tables to perform certain operations to make
sure that other services didn't mess with the data, and ocasionally the model
changed without you noticing.

Now, looking back, would I have leaned towards the same decision if I could go
back in time? Probably. One of the major lessons I learnt at REKKI was that best
practices are best practices for a reason. However, that reason also has a
context, and context is king in software development. Maybe if Netflix were to
decide to run that architecture it would bring them more grievances than joys,
however, for us it ticked the boxes that we needed ticked:

- low refactoring/migrations needed
- low-tech, low-maintenance
- high speed environment
- high self-sufficiency

## On structure

A different pattern I've already seen in multiple places is the usage of a
monorepos to contain most if not all of the company's services. However,
similarly to the previous example, this is one of the thing Sam Newman
discourages. He explains how he thinks that the best approach is to go with
different repositories and each repository have it's own test suite, CI
pipeline, etc. thus truly making services more independant. So, what makes it so
appealing to companies?

One of the points Sam does in his book to encourage a delayed adoption of
microservices is that sometimes the different domains aren't yet figured out.
Domains and solutions to problems emerge over time, like patterns, and by having
our code in a monolith we facilitate refactoring, while by leveraging
distributed services it becomes increasingly complex. However, what if all our
services were to live in the same repository? Suddenly refactoring becomes just
as easy. I believe this is one of the major advantages of a monorepo. Both the
extraction of code into libraries as well as the extraction of services
themselves becomes easier. And with this come a bunch of extra candy: service
discovery becomes easier, library upgrades and versioning becomes trivial and
cross-service pull requests are suddenly possible!

Not all are benefits though. Like Sam highlights, it also becomes very easy to
fudge the lines of different domains, thus defeating the purpouse of separated
services. And with that some tasks become more complex, such as CI pipelines,
tooling, etc.

So, why? Your mileage may vary, but In my experience the winner reasons are
usually dependency management and ease of change. The lines separating domains
may become fuzzier over time, but at the same time, refactoring those lines
becomes a non-issue. It stops becoming a set of different PRs which require
cross-team communication, but a single one which contains the whole thing. And,
at the end of the day, it makes folks more autonomous, independant.

Now, it's very easy to take the step from having a shared repository to making
mistakes like big-bang deploys and the like, since this would **truly** couple
services. But as long as we keep to the philosophy of one service, one pipeline,
one deploy, we're golden.

Many people I've had the chat with usually go with the ["but Google does it, so
it must work well"][6] argument. However, that does have many scaling issues
that must be handled.


## It's all about observability

I may be biased because it's a topic I'm truly fond of and enjoy, but in my
experience what makes or breaks a distributed system is how observable it is.
Usually if something goes wrong in a monolith, it's as easy as replicating the
underlying data, the original request, and reproducing. Your mileage may vary,
but it's fairly straight forward - only a small set of things may fail other
than the code. On the other hand, in a distributed system, the story is a
completely different one. Any service can fail the same way your monolith can
fail, but on top of that, all things communication can fail too.

If a user raises an issue that a purchase they're trying to do isn't coming
through, how can you know why and where it's failing? You can, of course start
by opening the logs on each of the services that the request goes through, and
try to follow the trail, but for example having all the logs aggregated helps
hugely, to start with. If you've previously done the work to make sure that a
user action has a correlation ID assigned and that correlation ID is passed down
to any downstream services, then it might be as easy as filtering our aggregated
logs by correlation ID. You can probably imagine how bloody it could get if the
request goes through more than a few services, you have to open the logs for
each in a new terminal session and on top of that you have no way of knowing
which requests are related.

Nonetheless, most of the times, the best place to start aren't logs. Depending
on the amount of trafic that our system has and how much we log, there are
better places to start looking: with a bit of luck, the system also has
distributed tracing in place. Different companies leverage tracing in different
ways. How I like thinking about it is as if a trace represented a _user story_:
That attempt of purchase by the user should be a trace in itself, with either 6
spans or 32 depending on where it failed.  This would mean that if he retried
the purchase 6 times, then there should be 6 traces for the same user. Now, if
our traces have enough metadata in them, we might be good to go by simply
filtering our traces by the user's email... and the traces will tell us exactly
what failed and where. Which was the first service that either timed out or
simply returned a 500? How did the upstream services react? And jumping from
this to the logs should be immediate since the trace would give us enough
context to know what we're looking for.

Some teams usually see observability as an afterthought: they finish their
features, ship them and then start instrumenting as things fail: if something
fails I instrument it. Ocasionally they may have some monitoring in place:
alerts on 500s or pages that take too long to load, but still not have
instrumented their system, which means that, yeah, you know when it fails...
but good luck finding the root cause. On the other hand, from experience, I
believe that instrumentation of systems should be in the core of the products
which we build. Folks talk about tests as a safety net when going to
production... but observability and monitoring is just the same. Good alerts let
you know when something is wrong early on, and good instrumentation let's you
fix it rapidly.

## Eventual consistency

I started this post talking about an unconventional architecture which I worked
with where services communicated through the database. One of the benefits it
had was that every operation in the system was transactional: every endpoint
opened a Postgres transaction, did it's magic and then commited, which meant
that we didn't have to deal with one of the pain points of distributed
architectures: eventual consistency.


We said that using the same DB as an integration point is bad. It creates
really bad coupling. the alternative is using _retry mechanisms_, queues, or
similar systems. This ensures **eventual consistency** VS the transactional
consistency that the DB offers us.

An alternative is to rollback changes, for example by deleting the created
resources. This does increase complexity greatly though.

How critical is our operation that we have to rely on actual transactions?

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

## Logging

Reading some [documentation from MSFT][5] some questions come to mind:

1. How is correlated logging handled? Do platforms like New Relic manage it out of the box
   or is it something k8s does? Does the developer do it?
2. Also, how is testing across services handled? Are other services stubbed, if so, how?
3. When a service fails, how far will the error bubble up? Also, how do you trace the error
back to it's original service if... the observability isn't there?

I can see how these cross-cutting concerns could be lifted to the API gateway, instead
of being handled directly by the actual services. Same as auth, etc.


## interesting reads

[Eventual Consistency Today: Limitations, Extensions, and Beyond][1]
[Reviews on the above paper][2]
[Consensus on Transaction Commit][4]



[0]: https://www.goodreads.com/book/show/22512931-building-microservices
[1]: https://queue.acm.org/detail.cfm?id=2462076
[2]: https://web.eecs.umich.edu/~mozafari/fall2018/eecs584/reviews/summaries/summary29.html
[3]: https://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf
[4]: https://arxiv.org/pdf/cs/0408036.pdf
[5]: https://docs.microsoft.com/en-us/azure/architecture/guide/architecture-styles/microservices
[6]: https://dl.acm.org/doi/fullHtml/10.1145/2854146
