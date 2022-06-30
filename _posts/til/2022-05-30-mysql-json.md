---
layout: post
title: "Working with JSON in MySQL"
author: Javier Garcia
category: mysql
tags: sql, mysql, json
---

Today I learnt how to work with JSON columns in MySQL. Previously I was familiar just with the Postgres way.

```sql
-- Accessing a value from a JSON column
SELECT attributes->'$.fee' FROM invoices WHERE attributes->'$.name' = 'purchase';

-- Is actually a shorthand for...
SELECT JSON_EXTRACT(attributes, '$.fee') FROM invoices WHERE JSON_EXTRACT(attributes, '$.name') = 'purchase';

-- If what you want to access is an array though:
SELECT attributes->'$.emails[0]' FROM invoices;

-- If you want to unquote the value, there's also a function for that
SELECT JSON_UNQUOTE(attributes->'$.name') FROM invoices;

-- Which also has a shorthand...
SELECT attributes->>'$.name' FROM invoices;
```

## Further handy functions

### `JSON_KEYS` to extract the keys

```
mysql> SELECT JSON_KEYS('{"a": 1, "b": {"c": 30}}');
+---------------------------------------+
| JSON_KEYS('{"a": 1, "b": {"c": 30}}') |
+---------------------------------------+
| ["a", "b"]                            |
+---------------------------------------+
```

### `JSON_SEARCH` to search document as a string

See the example below, but two important notes on this one:
- It behaves kinda like the SQL `LIKE`.
- It returns the MySQL JSON path syntax to access the found property. This means you'll have to do an extra query to find the actual value.

```
mysql> SET @j = '["abc", [{"k": "10"}, "def"], {"x":"abc"}, {"y":"bcd"}]';

mysql> SELECT JSON_SEARCH(@j, 'one', 'abc');
+-------------------------------+
| JSON_SEARCH(@j, 'one', 'abc') |
+-------------------------------+
| "$[0]"                        |
+-------------------------------+
```

## TLDR

- `->` operator allows for accessing JSON column properties
- `->>` is also for accessing properties, but also unquotes them.
- `[i]` square brackets allow you to access array values.
- You need to prepend the property you want to access with a `$.`. Kinda referencing the current column.

For further functionality... check docs: https://dev.mysql.com/doc/refman/8.0/en/json-search-functions.html
