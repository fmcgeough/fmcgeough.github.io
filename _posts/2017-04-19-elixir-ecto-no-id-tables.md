---
layout: post
title: Elixir, Ecto, Tables without id in Relationships
date: 2017-04-19 08:53:13
description: Using tables without id primary key in Ecto
categories: Elixir
tags: Elixir
---

Always, always, always use an "id" as the primary key for your relational database tables
where that "id" is a database sequence generated value. And if that
table has a relationship with another table then make sure your column name is:
"tablename_id". This is the common wisdom in using Ecto (or really any modern
relational library that attempts to help you with talking to your relational
database). But although there may be many people who tell you it can't be done any
other way that information is not true and the facts are laid out in the Ecto
documentation (which is really good as is the case with most of the Elixir library
doc that I've had to read).

The problem that I had to solve was that I had a lookup table that I called "instance_descriptions"
that stored static information on Amazon (AWS) EC2 instance types. Each instance type has a
wonderful natural key (A natural key is sometimes called a "domain key"). This natural key is
instance_type. If you've dealt with AWS then you are familiar with these.
They are strings like "c3.large" or "m4.xlarge".

In another table called "instances" I was storing all sorts of data retrieved via
the AWS API DescribeInstances. That data includes the instanceType. So, what I wanted
was a relationship that would work with Ecto queries between my "instances" and
"instance_descriptions" table based on the instance_type string column value.

I coded up my InstanceDescription module with something like this as the schema.

```
    @primary_key {:instance_type, :string, []}
    schema "instance_descriptions" do
      field :name, :string
      field :memory, :string
      field :vcpus, :string
      field :storage, :string

      has_many :instances, Instance

      timestamps()
    end
```

So, this is telling Ecto that I'm not using a generated "id" primary key but instead
using a column called "instance_type" that is a String. I'm declaring that there
may be many "instances" that are of any particular instance_type.

In the Instance module we have a bit more complicated expression to get Ecto to
perform the join properly. I've simplified the definition quite a bit by omitting all
the columns that store instance information and concentrating on just the relationship
columns.

```
    schema "instances" do
      field :instance_type, :string

      timestamps()

      belongs_to :instance_description, InstanceDescription, [foreign_key: :instance_type, references: :instance_type, define_field: false]
    end
```

You can see there is a lot of decoration that goes along with the belongs_to but that's
what you get if you want to vary off the default path. Here's how I understand what this is saying.
Our "instance_description" field is an InstanceDescription. It's foreign key field is going to use "instance_type"
and that foriegn key relationship (within InstanceDescription) is called "instance_type". I don't want
Ecto to generate a field for me. I already have one defined.

With all that in place you can now do :

```
    query = from i in Instance,
    left_join: d in assoc(i, :instance_description),
    preload: [instance_description: d]
    instances = Repo.all query
```

For anyone who stumbles upon this blog post I'd suggest you read the Ecto belongs_to]documentation.
It outlines what I described above.
