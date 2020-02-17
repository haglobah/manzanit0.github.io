---
layout: post
title: "Documenting Postgres tables and columns"
author: Javier Garcia
category: postgres
tags: postgres, sql, documentation
---

Today I learnt that you can document Postgres tables and columns!

```sql
CREATE TABLE people (
       id BIGSERIAL,
       full_name UUID NOT NULL,
       PRIMARY KEY (id),
);

COMMENT ON TABLE people IS 'This table contains all the people in the world';
COMMENT ON COLUMN people.full_name IS 'well, the name.';
```

And that to see the descriptions of columns/tables with [pgcli][0] you only have to type `\d+ people`, note the `+`.

The fantastic thing about this feature is that it makes it extremely easy to document some dodgy-named columns or tables,
in case the intention is not clear.

[0]: https://github.com/dbcli/pgcli
