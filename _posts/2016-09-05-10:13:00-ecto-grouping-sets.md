---
layout: post
title: Elixir/Ecto Postgresql Grouping sets
date: 2016-09-05 08:53:13
description: Exploring Features of Ecto - Elixir's Database Library
categories: Elixir
tags: Elixir
---

Sat down this morning and saw an interesting question on Elixir Ecto Slack
channel :

```
    Hi there! I need to build complex SQL queries in Elixir
    (that will rely on things like ROLLUP, CUBE, GROUPING
    SETS etc), yet I'm trying to figure out a way not to
    write those SQL queries by hand completely.
```

So that seemed like an interesting problem to look at since I'm
learning Ecto and so I tried to accomplish doing a grouping set
with straight Ecto. I didn't know if it was possible but as it turns
out, it is!

For an example, I used depesz (famous - to me - Postgresql community member who writes
great explanations of various Postgres features and functionality). His blog post is
at [https://www.depesz.com/2015/05/24/waiting-for-9-5-support-grouping-sets-cube-and-rollup/](https://www.depesz.com/2015/05/24/waiting-for-9-5-support-grouping-sets-cube-and-rollup/) explains a
bit about what those features are and why you might want to use them. The blog post
helpfully includes SQL to create a test table and SQL to exercise the ROLLUP and
CUBE. Since I was in the midst of working on my Dashboard project I just did a
temporary model in it to test things out.

Creating the table in Postgres is straightfoward:

```
    ▶ psql dashboard_dev
    psql (9.5.3)
    Type "help" for help.

    dashboard_dev=# create table test (
        d1 text,
        d2 text,
        d3 text,
        v int4,
        primary key (d1, d2, d3)
    );
```

Then inserting the test data is just as explained in the blog post :

```
    dashboard_dev=# insert into test
    select
        a, b, c,
        cast( random() * 50 as int4)
    from
        (values ('a'),('b') ) as d1(a),
        (values ('c'),('d') ) as d2(b),
        (values ('e'),('f') ) as d3(c)
    ;
```

I defined the Ecto schema as :

```
    defmodule Dashboard.Test do
      use Dashboard.Web, :model

      @primary_key false
      schema "test" do
        field :d1, :string, primary_key: true
        field :d2, :string, primary_key: true
        field :d3, :string, primary_key: true
        field :v, :integer
      end
    end
```

Notice that Ecto provides a means of defining a composite primary key.
The @primary_key false syntax seems like its stating that we won't be
using the usual generated sequence "id" column as a primary key. Then
you have to define all of the columns that make up the compound primary
key as "primary_key: true".

I had an .iex.exs file in the root of the application to provide help
for using iex. That file was :

```
    alias Dashboard.Repo
    alias Dashboard.{Account, User, Role, Test}

    import Ecto.Query, only: [from: 1, from: 2, subquery: 1, where: 2, first: 1]
```

Now starting up iex :

```
    iex(1)> q = from t in Test, group_by: fragment(" grouping sets( (d1), (d2,d3) )" ), select: [t.d1, t.d2, t.d3, sum(t.v)]
    #Ecto.Query<from t in Dashboard.Test,
     group_by: [fragment(" grouping sets( (d1), (d2,d3) )")],
     select: [t.d1, t.d2, t.d3, sum(t.v)]>
```

Well that's promising. Building the query didn't return errors. Lets try running it.

```
    iex(2)> Repo.all(q)
    [debug] QUERY OK db=3.7ms decode=0.1ms queue=0.1ms
    SELECT t0."d1", t0."d2", t0."d3", sum(t0."v") FROM "test" AS t0 GROUP BY  grouping sets( (d1), (d2,d3) ) []
    [["a", nil, nil, 99], ["b", nil, nil, 93], [nil, "c", "e", 58],
     [nil, "c", "f", 51], [nil, "d", "e", 57], [nil, "d", "f", 26]]
```

And that result is exactly right. Its what we'd get if we issued the sql in psql :

```
    dashboard_dev=#  select
        d1, d2, d3,
        sum(v)
    from
        test
    group by grouping sets ( (d1), (d2, d3) );
     d1 | d2 | d3 | sum
    ----+----+----+-----
     a  |    |    |  99
     b  |    |    |  93
        | c  | e  |  58
        | c  | f  |  51
        | d  | e  |  57
        | d  | f  |  26
    (6 rows)
```

Cube falls right out in the same manner.

```
    iex(3)> q = from t in Test, group_by: fragment(" cube( d1, d2, d3 )" ), select: [t.d1, t.d2, t.d3, sum(t.v)]
    #Ecto.Query<from t in Dashboard.Test,
     group_by: [fragment(" cube( d1, d2, d3 )")],
     select: [t.d1, t.d2, t.d3, sum(t.v)]>
    iex(4)> Repo.all(q)
    [debug] QUERY OK db=3.6ms decode=0.1ms
    SELECT t0."d1", t0."d2", t0."d3", sum(t0."v") FROM "test" AS t0 GROUP BY  cube( d1, d2, d3 ) []
    [["a", "c", "e", 12], ["a", "c", "f", 48], ["a", "c", nil, 60],
     ["a", "d", "e", 23], ["a", "d", "f", 16], ["a", "d", nil, 39],
     ["a", nil, nil, 99], ["b", "c", "e", 46], ["b", "c", "f", 3],
     ["b", "c", nil, 49], ["b", "d", "e", 34], ["b", "d", "f", 10],
     ["b", "d", nil, 44], ["b", nil, nil, 93], [nil, nil, nil, 192],
     ["a", nil, "e", 35], ["b", nil, "e", 80], [nil, nil, "e", 115],
     ["a", nil, "f", 64], ["b", nil, "f", 13], [nil, nil, "f", 77],
     [nil, "c", "e", 58], [nil, "c", "f", 51], [nil, "c", nil, 109],
     [nil, "d", "e", 57], [nil, "d", "f", 26], [nil, "d", nil, 83]]
```

Fragment is a powerful feature in Ecto. That was a fun problem and I was glad to learn
a bit more about what Ecto is capable of.
