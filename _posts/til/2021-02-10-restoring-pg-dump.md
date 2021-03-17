---
layout: post
title: Restoring a single Postgres schema
author: Javier Garcia
category: postgres
tags: sql, postgres
---

Today I learnt that `pg_dump` allows you to dump individual schemas, not just
the whole database, which is just great for multitenant scenarios leveraging
postgres schemas!

That said, here's a quick snippet of how you would backup the schema, and then
restore it:

```sh
# Dump the schema "tenant1"
pg_dump -c --host=production-database.com --port=5432 --username=postgres --dbname=cookbook_db --schema=tenant1 > backup_dump.sq

# Import the dump
psql --host=localhost --port=5432 --username=postgres --dbname=cookbook_db -f backup_dump.sql -L prod_dump_restore_attempt.log
```

Note: notice the `-c` option in `pg_dump`. That basically means the dump will
wipe clean the schema before importing it — while in some cases you might not
want do it it, I've found myself needing it most of the times to avoid
imcompatibilities.
