---
layout: post
title: Using Macros for Elxiir Unit Tests
date: 2024-06-19 11:30:00
description: When it is good to use a macro for a unit test
categories: elixir
tags: mix
giscus_comments: true
---

One of the strengths of the Elixir language is a powerful first-class macro language. The creator
of Elixir, José Valim, based on the development of Elixir macros (in part) on how Lisp macros work.
Macros are a form of metaprogramming. It's writing code that, in turn, writes code.

The unit test framework that is used for Elixir is [ExUnit](https://hexdocs.pm/ex_unit/ExUnit.html).
If you look at the ExUnit code you can see that it makes use of the macro language to define the
primitives used in Elixir's unit tests. For example, "describe" and "test" are both macros in the
ExUnit.Case module.

It's a very good thing to understand Elixir macros. If nothing else you will run into libraries
(like ExUnit) that make use of them (Ecto - the database ORM for Elixir - makes extensive use of
macros to define a DSL to simplify dealing with relational databases). However, once you learn
about Elixir macros its a very very good idea to limit your use of them. Although macros can
make a developer's life much easier they can also make it much more difficult by making the code
someone is trying to maintain exceedingly complex.

Once use of macros that I've used and have seen multiple developers use is to define a macro
for a test that defines the input used for the test itself. This means that instead of writing
n tests that all look very similar you can write one test which takes a list of n items and
generates n tests. The easiest way to understand this is with an example.

Suppose you have an Ecto.Schema for a database table in your app. It has some required fields.
If those fields are not provided when calling the `Ecto.Changeset` function `changeset/2` you
want the changeset to be invalid. I'll use a contrived example to demonstrate how I'd develop
that code and use a macro to generate unique tests. The examples is an `accounts` table.
We'll require that when building an `Ecto.Changeset` that `:account_id` and `account_state`
are required. All the other fields are optional. Here's the module defining the `Ecto.Schema`.

```
defmodule Account do
  @moduledoc """
  Ecto.Schema for the accounts table
  """
  use Ecto.Schema
  import Ecto.Changeset

  schema "accounts" do
    field(:account_id, :string)
    field(:account_state, :string)
    field(:url, :string)
    field(:owner_email, :string)
    field(:account_score, :integer)
    field(:current_billing_status, :string)
    field(:ui_schema, :string)

    timestamps()
  end

  @doc """
  What fields are used in this table (excludes default timestamp fields)?
  """
  def fields do
    [:account_id, :account_state, :url, :owner_email, :account_score, :current_billing_status, :ui_scheme]
  end

  @doc """
  What fields are required (not null) in this table?
  """
  def required do
    [:account_id, :account_state]
  end

  def changeset(struct, params \\ %{}) do
    struct
    |> cast(params, fields())
    |> validate_required(required())
  end
end
```

If we want to test that we get invalid Changesets when we don't provide the required fields we could have
a test like this:

```
defmodule AccountTest do
  use ExUnit.Case

  @valid_account_params %{account_id: "123", account_state: "active", url: "http://example.com", owner_email: "test@example.com"}

  for required_field <- Account.required() do
    test "error is generated when '#{required_field}' field is missing" do
      params = Map.delete(@valid_account_params, unquote(required_field))
      changeset = Account.changeset(%Account{}, params)
      refute changeset.valid?
      assert changeset.errors == [{unquote(required_field), {"can't be blank", [validation: :required]}}]
    end
  end
end
```

This lets us test each field independently. If the test fails it will tell us the name of the field that failed
(along with the failing line). For example, lets change the code to add in a field that is not required.

```
defmodule AccountTest do
  use ExUnit.Case

  # for required_field <- Account.required() do
  #   test "error is generated when '#{required_field}' field is missing" do
  #     params = Map.delete(@valid_account_params, unquote(required_field))
  #     changeset = Account.changeset(%Account{}, params)
  #     refute changeset.valid?
  #     assert changeset.errors == [{unquote(required_field), {"can't be blank", [validation: :required]}}]
  #   end
  # end

  for required_field <- Account.required() ++ [:url] do
    test "error is generated when '#{required_field}' field is missing" do
      params = Map.delete(@valid_account_params, unquote(required_field))
      changeset = Account.changeset(%Account{}, params)
      refute changeset.valid?
      assert changeset.errors == [{unquote(required_field), {"can't be blank", [validation: :required]}}]
    end
  end
end
```

When `mix test` is run we see the following:

```
  1) test error is generated when 'url' field is missing (AccountTest)
     test/account_test.exs:16
     Expected false or nil, got true
     code: refute changeset.valid?
     stacktrace:
       test/account_test.exs:19: (test)
```

The error line (19) is for the line `refute changeset.valid?` in our unit test file. The
first line tells us that the `url` field is what caused the problem.

There are a number of ways you can use macros to make your unit tests easier to read and maintain.
You should always keep the maintainability of the code in mind when you decide to start using
a macro.
