---
layout: post
title: Using Postgres COPY with Elixir/Ecto
date: 2017-09-29 08:53:13
description: Using Postgresql COPY with the Elixir ecto db library
categories: Elixir postgresql
tags: Elixir postgresql
---

This post is about using Postgres COPY to retrieve data from the database into memory when
using Elixir and Ecto. Let's assume you want to read all the data from a table 'accounts' using
Postgres COPY. Lets assume the table definition in Postgres is:

```
Table "public.accounts"
Column    |            Type             |                       Modifiers
-------------+-----------------------------+-------------------------------------------------------
id          | integer                     | not null default nextval('accounts_id_seq'::regclass)
name        | character varying(255)      |
address     | character varying(255)      |
city        | character varying(255)      |
state       | character varying(255)      |
zipcode     | character varying(255)      |
inserted_at | timestamp without time zone | not null
updated_at  | timestamp without time zone | not null
```

You can define an Ecto schema for that:

```
defmodule Account do
  use Ecto.Schema

  schema "accounts" do
    field :name, :string
    field :address, :string
    field :city, :string
    field :state, :string
    field :zipcode, :string
    field :inserted_at, :string
    field :updated_at, :string
  end
end
```

Since the data returned by COPY is String data the timestamps are defined in the schema as :string.

Reading the data is then:

```
columns = ["id", "name", "address", "city", "state", "zipcode", "inserted_at", "updated_at"]
stream = Ecto.Adapters.SQL.stream(Repo, "COPY (SELECT #{Enum.join(columns, ",")} FROM accounts) to STDOUT WITH CSV DELIMITER ',' ESCAPE '\"'")
{:ok, [result|_t]} = Repo.transaction(fn -> Enum.to_list(stream) end)
```

The data will be in result.rows. When we issued the COPY we explictly selected the order of the columns
in a SELECT statement supplied to COPY. The result.columns will be empty so we can't use that with COPY.
We also asked that the data be CSV with a escape of double-quote. This is necessary if we have newlines within
our data.

If we wanted to get into the Account struct to make it easier we'll want to use a CSV parsing library.
The [CSV Library](https://github.com/beatrichartz/csv) available from (hex.pm)[https://hex.pm/packages/csv]
will work fine for this. To convert the data into Account struct we could do the following:

```
cols = Enum.map columns, &(String.to_atom(&1))
a_result = result.rows
|> CSV.decode()
|> Enum.to_list()
|> Enum.filter(fn({result, _val}) -> result == :ok end)
|> Enum.reduce([], fn({:ok, a_list}, acc) -> [a_list | acc] end)
|> Enum.map(fn(row) -> struct(Account, Enum.zip(cols, row)) end)
```

This isn't something that you'd ordinarily choose to do. The reverse where you are using COPY to get data
into the database as quickly as possible is much more common. I just decided to write something up on this
because I hadn't seen anything that explained how to do this in Elixir. This was what I came up with but
I make no guarantees that there isn't a much easier way of doing this that I'm missing.
