---
layout: post
title: Use Postgresql 9.5 Upsert to Return Existing id
date: 2016-06-01 07:53:13
description: Returning the primary key id on upsert with Postgresql
categories: postgresql
tags: postgresql
---

Postgresql 9.5 introduced the long-waited for upsert. There are cases
where instead of actually updating when the row already exists what you
want is to return the primary key (or the whole row for that matter).
Postgresql has a SQL extension called RETURNING that lets you get the
values just inserted or updated but you can't just DO NOTHING and get
the value back.

Example :

```
      CREATE TABLE test_null_upsert(id serial,
                                    c2 character varying,
                                    constraint pk_null_upsert primary key(id),
                                    constraint unique_c2_upsert unique(c2));
      INSERT INTO test_null_upsert(c2) VALUES ('ABC');
```

Now if you attempt to insert 'ABC' it will fail and if you DO NOTHING
on the conflict you can't use RETURNING since there is nothing to return from doing nothing.

```
      INSERT into test_null_upsert(c2) values ('ABC') ON CONFLICT DO NOTHING RETURNING id;
      id
      ----
      (0 rows)

      INSERT 0 0
```

But if you perform a NULL update you can get your id column back.

```
      INSERT into test_null_upsert(c2) values ('ABC') ON CONFLICT ON CONSTRAINT unique_c2_upsert
      DO UPDATE SET c2 = test_null_upsert.c2 RETURNING id;

      id
      ----
      1
      (1 rows)

      INSERT 0 0
```
