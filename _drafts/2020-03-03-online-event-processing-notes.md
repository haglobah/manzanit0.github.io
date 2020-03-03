---
layout: post
title: Online Event Processing - Notes
author: Javier García
category: distributed systems
tags: distributed systems, notes
---

These are some notes when reading [Online Event Processing](https://queue.acm.org/detail.cfm?id=3321612)

# OLEP vs distributed transactions

## Distributed transactions

Nowadays polyglot persistence is a thing.

There are starting to be some databases which support distributed transactions
hence [google spanner][0] or voltdb

In case the different services use different db technologies, then you have
to rely on other type of approaches, like the 2-phase-commit.

On example of this is the [eXtended architecture][1]:

> The goal of XA is to guarantee atomicity in "global transactions" that are
> executed across heterogeneous components. A transaction is a unit of work such
> as transferring money from one person to another. Distributed transactions
> update multiple data stores (such as databases, application servers, message
> queues, transactional caches, etc.) To guarantee integrity, XA uses a two-phase
> commit (2PC) to ensure that all of a transaction's changes either take effect
> (commit) or do not (roll back), i.e., atomically.

The problem with heterogeneous transactions is that they always rely on the lowest
common denominator, hence they're usually problematic

Most search servers (ElasticSearch?) don't support distributed transactions, hence
leaving vulnerable to inconsistencies: 1) non atomic write, 2) different order of writes

If we have sytems write to a log, and the database systems read FROM the log, then consistency
is guaranteed.

## The log abstraction

Some implementations: Kafka, Pulsar, LogDevice.

It's:
-durable
-append-only
-sequential reads
-fault-tolerant
-partitioned

Subscribers may read events twice, but they never miss an event.

Some stream processors, like Apache Flink, support _exactly-once semantics_,
which means some kind of idempotency when processing. Nonetheless, if the
the service writes to an external storage, this cannot be ensured.


Check the diagram for ensuring atomicity in a payments system. It's quite enlightening.
It's relevant that there is only one payment executor thread per account, being it
sequential

### Some benefits

Since logs allow multiple subscribers, it allows different services to scale independently
as well as add/remove at will.

It is easier to ignore bad events than to undo DB writes

Debugging is easier with an append-only log than with a mutable db

With distributed transactions, if any one node is down, the transaction can't move forward. Not with logs.

### Some disadvantages

Even though there is an assurance of every event EVENTUALLY being consumed,
there is no upper bound on the time for it.  This gives place to **eventually
consistent systems**


## Further reads

- [Constraints in an environment empower the services](https://queue.acm.org/detail.cfm?id=2398392)
- [Evolution and Practice: Low-latency Distributed Applications in Finance](https://queue.acm.org/detail.cfm?id=2770868)
- [It Isn't Your Father's Realtime Anymore](https://dl.acm.org/doi/10.1145/1117389.1117409)
- [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)

[0]: https://cloud.google.com/spanner
[1]: https://en.wikipedia.org/wiki/X/Open_XA
