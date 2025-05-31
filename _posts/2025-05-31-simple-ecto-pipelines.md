---
layout: post
title: Simple Ecto query pipelines
date: 2025-05-31
description: How to use Ecto pipelines to allow flexibility in queries
tags:
categories: elixir
giscus_comments: true
---

The Ecto library is the means used by Elixir developers to interact with a relational database. One of the first things you might want to do when using Ecto is to query a database. This is a simple example showing how you can use a pipeline and a list of options to provide fairly robust querying capabilities for a single table.

## Example Table

Assume we have a "contacts" table in a Postgres database with an auto-generated id primary key.

```
   Column    |              Type
-------------+--------------------------------
 id          | bigint                         |
 first_name  | character varying(255)         |
 last_name   | character varying(255)         |
 inserted_at | timestamp(0) without time zone |
 updated_at  | timestamp(0) without time zone |
Indexes:
    "contacts_pkey" PRIMARY KEY, btree (id)
```

## Elixir Ecto Schema

The schema file in Elixir looks like this:

```
defmodule Ectyl.Database.Contact do
  use Ecto.Schema
  import Ecto.Changeset

  schema "contacts" do
    field :first_name, :string
    field :last_name, :string

    timestamps(type: :utc_datetime)
  end

  @doc false
  def changeset(contact, attrs) do
    contact
    |> cast(attrs, [:first_name, :last_name])
    |> validate_required([:first_name, :last_name])
  end
end
```

## Requirements

We want to write a module that lets developers:

- get all the contacts
- query on first name, last name or both to return only matching rows
- sort by first name, last name or both
- paginate

## Building a Pipelined Function

The way to do this is use a pipeline. Each step in the pipeline builds an Ecto.Query. We'll allow the caller to pass opts in to a function but provide a default that gives no opts.

```
defmodule Ectyl.Contacts do
  @moduledoc """
  The Contacts context.
  """

  import Ecto.Query, warn: false
  alias Ectyl.Database.Repo

  alias Ectyl.Database.Contact

  @contacts_fields Contact.__schema__(:fields)
  @directions [:asc, :desc]
  @operators [:like, :in, :not_in, :begins_with, :ends_with]

  @doc """
  Returns the list of contacts.

  ## Examples

      iex> Ectyl.Contacts.list_contacts()
      [%Contact{}, ...]

  ## Options
  - `:filters` - A list of filters to apply to the query. Each filter can be a tuple of `{field, value}` or `{field, value, operator}`.
    Supported operators are `:like`, `:in`, `:not_in`, `:begins_with`, and `:ends_with`. Supported fields are those defined in `Contact.__schema__(:fields)`
  - `:sort` - A list of tuples for sorting, where each tuple is `{field, direction}`. Supported fields are those defined in `Contact.__schema__(:fields)` and directions are `:asc` or `:desc`.
  - `:page` - The page number for pagination (default is 1).
  - `:page_size` - The number of contacts per page (default is 10).
  """
  def list_contacts(opts \\ []) do
    contacts_base_query()
    |> apply_filters(opts)
    |> apply_sorting(opts)
    |> apply_pagination(opts)
    |> Repo.all()
  end
```

## Building a Base Ecto.Query

The first step in the pipeline is to get the "base" `Ecto.Query`. This is the datatype that is passed to each function in the pipeline.

```
  defp contacts_base_query do
    from(c in Contact)
  end
```

## Applying Filters

The Ecto.Query built by that function is passed into the `apply_filters/2` function along with the `opts` provided by the caller:

```
  defp apply_filters(query, opts) do
    opts
    |> options(:filters)
    |> Enum.filter(fn
      {field, _value} when field in @contacts_fields -> true
      {field, _value, operator} when field in @contacts_fields and operator in @operators -> true
      _ -> false
    end)
    |> Enum.reduce(query, fn
      {field, value}, acc ->
        from c in acc, where: field(c, ^field) == ^value

      {field, value, :begins_with}, acc ->
        from c in acc, where: like(field(c, ^field), ^"#{value}%")

      {field, value, :ends_with}, acc ->
        from c in acc, where: like(field(c, ^field), ^"%#{value}")

      {field, value, :like}, acc ->
        from c in acc, where: like(field(c, ^field), ^"%#{value}%")

      {field, value, :in}, acc ->
        from c in acc, where: field(c, ^field) in ^value

      {field, value, :not_in}, acc ->
        from c in acc, where: field(c, ^field) not in ^value

      _, acc ->
        acc
    end)
  end
```

The `apply_filters/2` function extracts the `:filters` from the `opts` and defaults to an empty list. It then filters the list to ensure that the parameters are in the correct format. The check ensures that only fields that are in the schema are used. If an operator is used it must be one of the supported options. If an element in the list does not meet the criteria it is discarded. The function then uses an `Enum.reduce/4` where the `Ecto.Query` is used as an accumulator. Any alteration of the criteria builds a new `Ecto.Query` which becomes the updated accumulator.

## Applying Sorting

The `apply_sorting/2` function is similar.

```
  defp apply_sorting(query, opts) do
    opts
    |> options(:sort)
    |> Enum.filter(fn {field, direction} -> field in @contacts_fields and direction in @directions end)
    |> Enum.reduce(query, fn {field, direction}, acc ->
      from c in acc, order_by: [{^direction, field(c, ^field)}]
    end)
  end
```

The function extracts the `sort` options and defaults to an empty list. It then filters the list to ensure parameters are in the right format. Again, the `Ecto.Query` is used as the accumulator and sorts added build an updated `Ecto.Query` accumulator.

## Apply Pagination

The final function in the pipeline is `apply_pagination`. In this case the function will use pagination regardless of whether or not the caller specified `:page` or `:page_size`.

```
  defp apply_pagination(query, opts) do
    page = opts[:page] || 1
    page_size = opts[:page_size] || 10

    query
    |> offset(^((page - 1) * page_size))
    |> limit(^page_size)
  end
```

## Execute Query

The final step is to pass the `Ecto.Query` to `Repo.all` to return results. In this case the list contains `[%Contact{}, ...]`.

## Problems with this Approach

This is a trivial example. Although it is providing fairly robust capabilities (filters, sorting, pagination) it falls apart if we want to join to other tables (for example, lets assume there's a phone_numbers table that has a foreign key to contacts to allow a contact to have any number of phone numbers). The reason this probably does not work is that we might want to filter on the phone number (for example, give me all the contacts in area code 678) but there's no means of indicating what table a filter applies to. There's a similar situation with sorting. I'll create another post to indicate how that might be handled.

There is some checking of the parameters provided but there is no notification (logging, telemetry) to indicate that the function received bad input. In most cases you would want to have that. I think it's generally fine to have the code proceed if the input data can be cleaned up but I'd want to know that some part of the codebase is passing input data that will never work.

## Benefits of this Approach

Pipelining to build an `Ecto.Query` is a powerful tool that lets you write modules that provide filtering, sorting and pagination in a way that is fairly powerful while being easy to read and maintain.
