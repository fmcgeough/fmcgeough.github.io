---
layout: post
title: Elixir, Phoenix and Ecto fragment and SIMILAR TO
date: 2017-10-03 08:53:13
description: Using regex style capabilities in Elixir with Ecto
categories: Elixir
tags:
---

I had a part of a Phoenix web app where users wanted to save filters
to the database and then be able to select and apply them to narrow
a dataset. One of the key features that they wanted was the ability to
wildcard search the fqdn column value (fqdn is fully qualified domain
name in this case). Creating the filter and saving the data was fairly
straightfoward but how could I use Ecto to apply "n" number of LIKE
operators?

The fqdn filters may or may not be there so I knew I needed to build
up an Ecto query with a function. If there are no fqdn filters then
I'd just return the original query. I knew that if there were fqdn_filters
I needed to "AND" this part of the query with all the rest of the criteria
but I needed to "OR" between as many fqdn filters as a user defined.

Postgres has a neat feature called "SIMILAR TO" that applies in this case.
You can read about it under [Pattern Matching](https://www.postgresql.org/docs/current/functions-matching.html).

As they note in their doc: The SIMILAR TO operator returns true or false depending on whether its pattern matches the given string. It is similar to LIKE, except that it interprets the pattern using the SQL standard's definition of a regular expression. SQL regular expressions are a curious cross between LIKE notation and common regular expression notation.

So with this knowledge and Ecto's fragment feature I solved the problem. One issue
remained. How do I specify the column properly in the fragment. Thankfully this is
provided in Ecto as well. I just used a substitution parameter for the column name
itself. This allows Ecto to resolve the alias properly.

```
# Add in FQDN filters if specified
def add_fqdn_filters(query, fqdn_filters) when fqdn_filters == nil, do: query
def add_fqdn_filters(query, fqdn_filters) do
  filter_for_pg = build_pg_filter(fqdn_filters)
  from i in query,
  where: fragment("? SIMILAR TO ?",  i.fqdn, ^filter_for_pg)
end

# The PostgreSQL SIMILAR operator requires a string in
# a particular format.
defp build_pg_filter(fqdn_filters) do
  all_filters = Enum.join(fqdn_filters, "|")
  "%(" <> all_filters <> ")%"
end
```

Its key to note that doing this type of criteria has performance implications. There is no
way for Postgresql to use an index to resolve the query. In this case it was fine because
the actual dataset was less than 10,000 rows and other parts of the criteria were more
amenable to index usage and limiting the result set.
