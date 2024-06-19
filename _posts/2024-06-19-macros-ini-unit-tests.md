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
of Elixir, José Valim, based the development of Elixir macros (in part) on how Lisp macros work.
Macros are a form of metaprogramming. It's writing code that, in turn, writes code.

The unit test framework that is used for Elixir is [ExUnit](https://hexdocs.pm/ex_unit/ExUnit.html).
If you look at the ExUnit doc you can see that it makes use of the macro language to define the
primitives used in Elixir's unit tests. For example, "describe" and "test" are both macros in the
ExUnit.Case module. Here is how `describe` is defined (as of June 2024). You can see it uses
`defmacro`. Its interesting (and probably a bit confusing) when you recognize that defmacro
is also a macro.

```
  defmacro describe(message, do: block) do
    definition =
      quote unquote: false do
        defp unquote(name)(var!(context)), do: unquote(body)
      end

    quote do
      ExUnit.Callbacks.__describe__(__MODULE__, __ENV__.line, unquote(message), fn
        message, describes ->
          res = unquote(block)

          case ExUnit.Callbacks.__describe__(__MODULE__, message, describes) do
            {nil, nil} -> :ok
            {name, body} -> unquote(definition)
          end

          res
      end)
    end
  end
```

It's a good idea to get at least some familiarity with Elixir macros. If nothing
else you will run into libraries (like ExUnit, Ecto) that make use of them and
the Elixir language itself defines many functions using macros. Ecto, the database
ORM for Elixir, makes extensive use of macros to define a DSL to
simplify dealing with relational databases.

Once you learn about Elixir macros its a very good idea, in general, to not use
them. Although well-done macros can make a developer's life much easier they
also can make code way more obtuse. This raises the cost of maintaining the
software.

One use of macros that you might see (or use yourself) is to define a macro for
a test that defines the input used for the test itself. This means that instead
of writing n tests that all look very similar you can write one test which takes
a list of n items and generates n tests. The easiest way to understand this is
with an example.

Suppose you have an Ecto.Schema for a database table in your app. It has some
required fields. If those fields are not provided when calling the
`Ecto.Changeset` function `changeset/2` you want the changeset to be invalid.
I'll use a contrived example to demonstrate how I'd develop that code and use a
macro to generate unique tests. The example is an `accounts` table. We'll
require that when building an `Ecto.Changeset` that `:account_id` and
`account_state` are required. All the other fields are optional. Here's the
module defining the `Ecto.Schema`.

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
$ mix test
  1) test error is generated when 'url' field is missing (AccountTest)
     test/account_test.exs:16
     Expected false or nil, got true
     code: refute changeset.valid?
     stacktrace:
       test/account_test.exs:19: (test)
```

The error line (19) is for the line `refute changeset.valid?` in our unit test
file. The first line tells us that the `url` field is what caused the problem.

This same unit test could be rewritten easily without this approach. But let's
add in the same issue as the "bad" test and check the output when `mix test` is
run.

```
defmodule AccountTest do
  use ExUnit.Case

  @valid_account_params %{account_id: "123", account_state: "active", url: "http://example.com", owner_email: "test@example.com"}

  test "error is generated when a required field is missing" do
    for required_field <- Account.required() ++ [:url] do
      params = Map.delete(@valid_account_params, required_field)
      changeset = Account.changeset(%Account{}, params)
      refute changeset.valid?
      assert changeset.errors == [{required_field, {"can't be blank", [validation: :required]}}]
    end
  end
end
```

When `mix test` is run we see the following:

```
$ mix test

  1) test error is generated when a required field is missing (AccountTest)
     test/account_test.exs:15
     Expected false or nil, got true
     code: refute changeset.valid?
     stacktrace:
       test/account_test.exs:19: anonymous fn/2 in AccountTest."test error is generated when a required field is missing"/1
       (elixir 1.14.2) lib/enum.ex:2468: Enum."-reduce/3-lists^foldl/2-0-"/3
       test/account_test.exs:16: (test)
```

With that output its not possible to see the field that caused the failure. We
can change the `refute` line to `refute changeset.valid?, "Removing
'#{required_field}' didn't make changeset invalid"` so that the field is output
when that line fails. That gives us:

```
  1) test error is generated when a required field is missing (AccountTest)
     test/account_test.exs:15
     Removing 'url' didn't make changeset invalid
     code: for required_field <- Account.required() ++ [:url] do
     stacktrace:
       test/account_test.exs:19: anonymous fn/2 in AccountTest."test error is generated when a required field is missing"/1
       (elixir 1.14.2) lib/enum.ex:2468: Enum."-reduce/3-lists^foldl/2-0-"/3
       test/account_test.exs:16: (test)
```

So that output tells us what we want to know. When the `url` field is removed it
does not make the changeset invalid.

There are a number of ways you can use macros to make your unit tests easier to
read and maintain. You should always keep the maintainability of the code in
mind when you decide to start using a macro. When you are using a macro it's
always possible to rework the code to not use it. You have to decide if using
the macro makes the code more maintainable or less.
