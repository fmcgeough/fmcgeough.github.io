---
layout: post
title: Elxiir Unit Tests and Iterating a Single Test
date: 2024-06-19 11:30:00
description: What does it mean when I have a test block in Elixir?
categories: elixir
tags: ex_unit
giscus_comments: true
---

One of the strengths of the Elixir language is a powerful first-class macro
language. The creator of Elixir, José Valim, based the development of Elixir
macros (in part) on how Lisp macros work. Macros are a form of metaprogramming.
It's writing code that, in turn, writes code.

It's a good idea to get at least some familiarity with Elixir macros. If nothing
else you will run into libraries (like ExUnit, Ecto) that make use of them and
the Elixir language itself defines many functions using macros. Ecto, the
database ORM for Elixir, makes extensive use of macros to define a DSL to
simplify dealing with relational databases.

## the test macro in the ex_unit library

The unit test framework that is used for Elixir is
[ExUnit](https://hexdocs.pm/ex_unit/ExUnit.html). If you look at the ExUnit doc
you can see that it makes use of the macro language to define the primitives
used in Elixir's unit tests. For example, "describe" and "test" are both macros
in the ExUnit.Case module. Here is how `test` is defined (as of June 2024). You
can see it uses `defmacro`. Its interesting (and probably a bit confusing) when
you recognize that defmacro is also a macro.

```
  defmacro test(message) do
    %{module: mod, file: file, line: line} = __CALLER__

    quote bind_quoted: binding() do
      name = ExUnit.Case.register_test(mod, file, line, :test, message, [:not_implemented])
      def unquote(name)(_), do: flunk("Not implemented")
    end
  end
```

## Breaking down the test macro

Let's consider (in general) how macros work with Elixir. When your code is
compiled it is converted to an AST (this is an abstract syntax tree. An AST is
your code converted to data). The places where the code is actually a macro are
identified. Once the AST is generated the macro is expanded. This can be a
recursive process. But eventually everything is converted to an AST. At that
point the AST can be converted to the byte code that executes in the VM.

In the `test` macro above a signle parameter is passed. This is the name of the
unit test. The first thing the test macro does is use the `__CALLER__` macro to
get the module running the test, its file, and the line number of the test.

Then `test` does a quote block. As you maybe can guess `quote` is also a macro. It takes
two arguments: `opts` and `block`. The `opts` in this case are `bind_quoted:
binding()`. This option passes a binding to the macro. Whenever a binding is
given, `unquote/1` is automatically disabled. The call `binding(context \\ nil)`
returns the binding for the given context as a keyword list. In the returned
result, keys are variable names and values are the corresponding variable
values. If the given context is nil (by default it is and it is in how the test
macro is written), the binding for the current context is returned.

Inside the do block a call is made to `ExUnit.Case.register_test`. This is what
will check that the name of your test is actually unique. It's probably good
to point out that the String that you use for your test can be a maximum of
255 characters. If you give a String longer than that you'll get a compile
error that looks something like this:

```
== Compilation error in file test/account_test.exs ==
** (SystemLimitError) the computed name of a test (which includes its type, the name of its parent describe block if present, and the test name itself) must be shorter than 255 characters
```

## using a comprehension above a test

One use of macros that you might see (or use yourself) is to define a comprehension
outside a test declaration. This allows you to write a single test that uses
each of the comprehension values. So, instead of writing a number of unit tests
that end up having the same code you write the test once and make it data-driven.
The easiest way to understand this is with an example.

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

So, what is going on here? Well, the comprehension takes each value in turn and
uses that as input to the `test` macro. The `required_field` value becomes
available in the unit test string and in the test block itself. In the test
string `"error is generated when '#{required_field}' field is missing"` no
interpolation is needed. This is because the string interpolation is inside
the context of the macro itself. The test block executes in its own context
so you need to use `unquote(required_field)` to get actual value. If you
tried to just use `required_field` you'll get a compiler error.
`undefined function required_field/0`.

If you wanted you could output the binding just before the `test` line:

```
  for required_field <- Account.required() do
    IO.inspect(binding(), label: "binding")
    test "error is generated when '#{required_field}' field is missing" do
      params = Map.delete(@valid_account_params, unquote(required_field))
      changeset = Account.changeset(%Account{}, params)
      refute changeset.valid?
      assert changeset.errors == [{unquote(required_field), {"can't be blank", [validation: :required]}}]
    end
  end
```

Since we have two required fields (`:account_id` and `:account_state`) this outputs:

```
binding: [required_field: :account_id]
binding: [required_field: :account_state]
```

## Rewrite without the comprehension

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

## Recap

Elixir macros are very powerful. You should not write a macro without thinking
through a couple of things:

- are you sure you are going to be able to maintain the macro. Is the macro going
  to be maintainable for other developers?
- are you sure that the macro makes developers lives easier long term? Could you
  get the same value from just a regular function?

Using a comprehension to pass data into a test to avoid repeating code by making
the test data-driven is a technique that you might want to consider if it meets
your use case.
