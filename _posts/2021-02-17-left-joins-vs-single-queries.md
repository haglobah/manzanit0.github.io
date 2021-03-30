---
layout: post
title: Deciding between LEFT JOINs and separate single queries
author: Javier Garcia
category: postgres
tags: sql, postgres
---

I've always believed that whenever you managed to retrieve the data in a single
query, that would always be more optimal than doing multiple queries, just
because there would be less round-trips, because Postgres wouldn't overfetch...
whatever. Well, I was wrong.

Today I learnt that in the case of LEFT JOINs, sometimes it's much more
expensive to join than to do separate queries. Why? The reason is that when you
do a left join between table A and table B you get all the records of table A
repeated as many times as children it has in table B. This means that **a LEFT
JOIN will use exponentially more data since the result will be a cartesian
product**.

This sounds terribly mathematic and cryptic, and could probably be summarised
in something along the lines of "a single left join is good, but if you're
going to use more than one, then measure against alternative solutions before
deploying", but let's shed some light into it with an example.

## Show, don't tell

Say we have an application which processes posts, and those posts have both
comments and tags related to them:

```
+------------+     +---------+     +---------+
|  comment   |     |   post  |     |  tag    |
|------------|*   1|---------|1   *|---------|
| id         |-----| id      |-----| id      |
| post_id    |     | ...     |     | post_id |
| ...        |     |         |     | ...     |
+------------+     +---------+     +---------+
```

We can create such a model in our local database with the following script:


```sql
-- setup.sql

CREATE DATABASE playground;

BEGIN;

CREATE TABLE posts (
  id serial NOT NULL,
  PRIMARY KEY(id)
);

CREATE TABLE comments (
  id serial NOT NULL,
  post_id int NOT NULL,
  PRIMARY KEY(id),
  FOREIGN KEY (post_id) REFERENCES posts(id)
);

CREATE TABLE tags (
  id serial NOT NULL,
  post_id int NOT NULL,
  PRIMARY KEY(id),
  FOREIGN KEY (post_id) REFERENCES posts(id)
);

-- Three posts
INSERT INTO posts (id) VALUES (1), (2), (3);

-- Three coments per post
INSERT INTO comments (id, post_id) VALUES (1, 1), (2, 1), (3, 1), (4, 2), (5, 2), (6, 2), (7, 3), (8, 3), (9, 3);

-- Three tags per post
INSERT INTO tags (id, post_id) VALUES (1, 1), (2, 1), (3, 1), (4, 2), (5, 2), (6, 2), (7, 3), (8, 3), (9, 3);

COMMIT;
```

Notice that we have 3 posts, with IDs 1, 2 and 3, and each post has three
comments with IDs 1, 2 and 3, as well as three tags also with IDs 1, 2 and 3.

Say we want to fetch all our posts along with their comments. If we were to do
a single LEFT JOIN to get posts and comments, the query and its results would
look like below:


```
playground> SELECT posts.id as post_id, comments.id as comment_id
            FROM posts
            LEFT JOIN comments on posts.id = comments.post_id;

+-----------+--------------+
| post_id   | comment_id   |
|-----------+--------------|
| 1         | 1            |
| 1         | 2            |
| 1         | 3            |
| 2         | 4            |
| 2         | 5            |
| 2         | 6            |
| 3         | 7            |
| 3         | 8            |
| 3         | 9            |
+-----------+--------------+
SELECT 9
Time: 0.024s
```

So far, so good: we got the data we needed without any redundancy really. If we
had done separate queries we would have gotten 3 posts and 9 comments: 12
rows... So we can argue that it even makes more sense to do a LEFT jOIN in
terms of the amount of data fetched.

However, now say that we also add the tags to the mix and also LEFT JOIN on those:

```
playground> SELECT posts.id as post_id, comments.id as comment_id, tags.id as tag_id
            FROM posts
            LEFT JOIN comments on posts.id = comments.post_id
            LEFT JOIN tags on tags.post_id = posts.id;

+-----------+--------------+----------+
| post_id   | comment_id   | tag_id   |
|-----------+--------------+----------|
| 1         | 3            | 1        |
| 1         | 2            | 1        |
| 1         | 1            | 1        |
| 1         | 3            | 2        |
| 1         | 2            | 2        |
| 1         | 1            | 2        |
| 1         | 3            | 3        |
| 1         | 2            | 3        |
| 1         | 1            | 3        |
| 2         | 6            | 4        |
| 2         | 5            | 4        |
| 2         | 4            | 4        |
| 2         | 6            | 5        |
| 2         | 5            | 5        |
| 2         | 4            | 5        |
| 2         | 6            | 6        |
| 2         | 5            | 6        |
| 2         | 4            | 6        |
| 3         | 9            | 7        |
| 3         | 8            | 7        |
| 3         | 7            | 7        |
| 3         | 9            | 8        |
| 3         | 8            | 8        |
| 3         | 7            | 8        |
| 3         | 9            | 9        |
| 3         | 8            | 9        |
| 3         | 7            | 9        |
+-----------+--------------+----------+
SELECT 27
Time: 0.013s
```

In this case we got 27 rows... while if we had done separate queries we would
have gotten 3 posts, 9 comments and 9 tags making it a total of 21 rows too. We
can start to see the difference, even if it's very subtle in this particular
case.


## Taking it to the next level: Increasing volume

To really highlight the point let's beef up the amount of records. Take this
little script to insert 100.000 posts and 3 tags and coments per post.


```sql
-- Wipe the tables in case you've been tampering around with the records.
DELETE FROM tags;
DELETE FROM comments;
DELETE FROM posts;

-- Insert 100.000 posts.
INSERT INTO posts(id) SELECT (generate_series(1, 100000)) ON CONFLICT DO NOTHING;

-- Insert 3 comments and tags per post
DO
$$
DECLARE f RECORD;
DECLARE count NUMERIC := 1;

BEGIN
    FOR f IN SELECT id FROM posts LOOP
      FOR n IN 1..3 LOOP
        INSERT INTO comments(id, post_id) VALUES (count, f.id) ON CONFLICT DO NOTHING;
        INSERT INTO tags(id, post_id) VALUES (count, f.id) ON CONFLICT DO NOTHING;
        count := count +1;
      END LOOP;
    END LOOP;
END;
$$
```

After running the script we'll find ourselves with 100.000 posts and 300.000 tags and comments, so if
we re-run the above queries... let's check out the results:

```
playground> SELECT count(*)
            FROM posts
            LEFT JOIN comments on posts.id = comments.post_id;
+---------+
| count   |
|---------|
| 300000  |
+---------+
SELECT 1
Time: 0.083s

playground> SELECT count(*)
            FROM posts
            LEFT JOIN comments on posts.id = comments.post_id
            LEFT JOIN tags on tags.post_id = posts.id;
+---------+
| count   |
|---------|
| 900000  |
+---------+
SELECT 1
Time: 0.189s
```


Now we're talking. With a single LEFT JOIN, we're retrieving 300.000 rows. In
other words, 300.000 comments with the post data in each row. If we were to
have done it in single queries, we would have retrieved a total of 100.000
posts and 300.000 comments, so again, a LEFT JOIN is a good option. However, in
the second scenario, we've retrieved 900.000 rows. With separate queries we
would have retrieved 300.000 comments, 300.000 tags and 100.000 posts. This
means that by left joining both tables we're retreiving 200k records which
othrwise we would not need! Think of this in terms of the work we're loading
into thed database and, in case you're not streaming the data, the amount of
memory you'll load the server with.

## Final thoughts

As we've seen, not thinking through usage of a LEFT JOIN can end up being way
less performant than shooting separate queries. At the end of the day, it all
comes back to the mantra any developer should have: _always measure_.