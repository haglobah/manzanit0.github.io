---
layout: post
title: "Working with Postgres arrays"
author: Javier García
category: postgres
tags: postgres, sql
---

Today I had to do some data cleaning at Rekki and in an attempt to not have to
do things manually one record at a time or code a script, I looked at how to do
it with SQL... to my luck, I found Postgres offers some powerful array
manipulation functions.

These are some handy formulas for adding/replacing/removing array data in
Postgres:

```sql
-- Finding records with a certain value in the array
SELECT id, name, tags FROM suppliers WHERE 'integrated' = ANY(tags)

-- Adding a coupleof values to the array
UPDATE suppliers SET topics = array_cat(tags, '{integration_1, integration_2}');

-- Removing a value from the array
UPDATE suppliers SET tags = array_remove(tags, 'integrated');

-- Replacing a value from the array
UPDATE suppliers SET tags = array_replace(topics, 'integrated', 'not_integrated');
```
