---
layout: post
title: Rename inserted_at column in Elixir/Ecto
date: 2017-10-03 08:53:13
description: Modifying the name of a built-in column in Elixir/Ecto
categories: Elixir
tags: Elixir
---

By default Ecto can generate the time a row was inserted or updated
in your database. By default these columns are called:

```
- inserted_at
- updated_at
```

Developers coming from Rails might want to use created_at instead of
inserted_at. This is easy to do. Lets say you want a table "tests"
defined as:

```
id         | bigint                      | not null default nextval('tests_id_seq'::regclass)
name       | character varying(255)      |
created_at | timestamp without time zone | not null
updated_at | timestamp without time zone | not null
```

In your migration file you'll want to use:

```
def change do
  create table(:tests) do
    add :name, :string

    timestamps(inserted_at: :created_at)
  end
end
```

So you are passing parameters to the timestamps function to let it know
that inserted_at is represented by the field created_at.

Then in your schema file you'll want to do the same:

```
schema "tests" do
  field :name, :string

  timestamps(inserted_at: :created_at)
end
```

That's it. Everything will work and created_at will be populated when
you insert rows with Ecto.
