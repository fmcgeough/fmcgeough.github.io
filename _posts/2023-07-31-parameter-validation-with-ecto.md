---
layout: post
title: Elixir, Parameter Validation with Ecto
date: 2023-07-31 08:53:13
description: Using the Ecto library to validate API Parameters
categories: elixir
tags: validation api
giscus_comments: true
---

Elixir is a language that first appeared in 2012. There is a web framework called Phoenix
that is what is primarily used to develop web-based software in Elixir. The Ecto library
is used in Elixir to interact with relational databases. However, Ecto can also be used
to validate parameters passed to API endpoints. This is a quick example to help you
understand how this is done. It assumes you are already at least somewhat familiar with
Elixir and the Phoenix framework. The example is just that, an example. It's not complete.
It's not meant to be an example of production-level code. You can think of this as an
intro and getting started type of thing. It's got enough complexity to get you past some
initial hurdles.

## Prerequisites

The sample code is in a Phoenix framework app generated with:

```
mix phx.new --no-assets  --no-dashboard --no-ecto --no-html --no-live --no-mailer --no-tailwind paramz
```

Then the Ecto dependency was added to the mix.exs file:

```
{:ecto, "~> 3.10"}
```

It is not necessary to add a dependency on `ecto_sql` nor `db_connection`. In an actual application where
you have a dependency on `ecto_sql` it is not necessary to add a dependency on `ecto`. The dependency on
`ecto_sql` brings in the `ecto` library.

## Router

For the example code we have a router with two endpoints:

- a simple endpoint with parameters in the path
- a more complex endpoint with parameters in path, a JSON body containing a nested object and a query string.

```
defmodule ParamzWeb.Router do
  use ParamzWeb, :router

  pipeline :api do
    plug(:accepts, ["json"])
  end

  scope "/api", ParamzWeb do
    pipe_through(:api)

    # Simple request with parameters in path
    get("/:account_id/:user_id/simple", ParamCheckController, :simple)

    # More complex request with parameter in path, a body containing
    # data and allows a query string
    post("/:account_id/complex", ParamCheckController, :complex)
  end
end
```

## Controller Functions

For both our controller functions the example code just validates the parameters
and then sends back 200 and the validated parameters (which have conversions
done). If the parameters are invalid the code returns a 400 status code with the
error info on the parameters supplied.

```
defmodule ParamzWeb.ParamCheckController do
  use ParamzWeb, :controller

  import ParamzWeb.Params

  alias ParamzWeb.Params.{Complex, Simple}

  def simple(conn, params) do
    params
    |> Simple.validate()
    |> handle_validation_result()
    |> case do
      {:ok, validated_params} ->
        IO.inspect(validated_params, label: "after Params call")
        conn |> put_status(200) |> json(Map.from_struct(validated_params))

      {:error, :invalid_input, error_msg} ->
        conn |> put_status(400) |> json(%{error: "Invalid parameter input. #{error_msg}"})
    end
  end

  def complex(conn, params) do
    params
    |> Complex.validate()
    |> handle_validation_result()
    |> case do
      {:ok, validated_params} ->
        IO.inspect(validated_params, label: "after Params call")
        conn |> put_status(200) |> json(validated_params)

      {:error, :invalid_input, error_msg} ->
        conn |> put_status(400) |> json(%{error: "Invalid parameter input. #{error_msg}"})
    end
  end
end
```

## Parameter Validation Code

### General Utility

A simple module to convert a changeset into either {:ok, map()} or {:error, :invalid_input, String.t()}

```
defmodule Paramz.Utils.LogHelp do
  @moduledoc """
  Utility functions related to logging errors
  """

  alias Ecto.Changeset

  @doc """
  If our app code gets back an error tuple from Ecto where the second
  element is an Ecto.Changeset this function can convert that value
  into a String for logging
  """
  @spec ecto_error(Ecto.Changeset.t()) :: String.t()
  def ecto_error(%Ecto.Changeset{} = changeset) do
    IO.inspect(changeset, label: "changeset errors")

    Changeset.traverse_errors(changeset, fn {msg, opts} ->
      Enum.reduce(opts, msg, fn {key, value}, acc ->
        String.replace(acc, "%{#{key}}", _to_string(value))
      end)
    end)
    |> Enum.reverse()
    |> Enum.map_join(", ", fn {k, msgs} ->
      "#{k}: #{_to_string(msgs)}"
    end)
  end

  def ecto_error(changeset) do
    raise ArgumentError, "unsupported argument: #{inspect(changeset)}"
  end

  defp _to_string(msgs) when is_list(msgs) do
    msgs |> Enum.map_join(", ", &_to_string(&1))
  end

  defp _to_string(msg) when is_binary(msg) do
    String.replace(msg, "\\n", " ")
  end

  defp _to_string(msg) do
    "#{inspect(msg)}"
  end
end
```

### Validation for the simple endpoint

This endpoint has two parameters in the path. One is the account id which is an
integer. The other is the user id which is also an integer.

```
defmodule ParamzWeb.Params.Simple do
  use Ecto.Schema
  import Ecto.Changeset

  @max_int32 (:math.pow(2, 31) - 1) |> round()

  @primary_key false
  embedded_schema do
    field(:account_id, :integer)
    field(:user_id, :integer)
  end

  def validate(attrs) do
    changeset(%__MODULE__{}, attrs)
  end

  def changeset(%__MODULE__{} = struct, attrs \\ %{}) do
    struct
    |> cast(attrs, [:account_id, :user_id])
    |> validate_required([:account_id, :user_id])
    |> validate_number(:account_id, greater_than: 0, less_than_or_equal_to: @max_int32)
    |> validate_number(:user_id, greater_than: 0, less_than_or_equal_to: @max_int32)
  end
end
```

## Doing validation for complex endpoint

In this endpoint there’s a parameter in the path - the account id - a body
containing JSON with a nested object - and a query parameter.

In order to handle nested objects in Ecto validation you can use the embeds_one
syntax. This means that every nested object needs its own Ecto.Schema defined
and has its own separate changeset functionality.

```
defmodule ParamzWeb.Params.License do
  use Ecto.Schema
  import Ecto.Changeset

  @max_int32 (:math.pow(2, 31) - 1) |> round()

  @primary_key false
  embedded_schema do
    field(:id, :integer)
    field(:type, :string)
  end

  def changeset(struct, attrs) do
    struct
    |> cast(attrs, [:id, :type])
    |> validate_required([:id, :type])
    |> validate_number(:id, greater_than: 0, less_than_or_equal_to: @max_int32)
  end
end

defmodule ParamzWeb.Params.License do
  use Ecto.Schema
  import Ecto.Changeset

  @max_int32 (:math.pow(2, 31) - 1) |> round()

  @primary_key false
  embedded_schema do
    field(:id, :integer)
    field(:type, :string)
  end

  def changeset(struct, attrs) do
    struct
    |> cast(attrs, [:id, :type])
    |> validate_required([:id, :type])
    |> validate_number(:id, greater_than: 0, less_than_or_equal_to: @max_int32)
  end
end

defmodule ParamzWeb.Params.Complex do
  use Ecto.Schema
  import Ecto.Changeset

  alias ParamzWeb.Params.License

  @max_int32 (:math.pow(2, 31) - 1) |> round()
  @allowed_includes ["posts", "pictures"]
  @allowed_includes_str Enum.join(@allowed_includes, ", ")
  @invalid_include_value "invalid include values. Allowed values:"
  @invalid_include_format "invalid include value. Must be list"

  @primary_key false
  embedded_schema do
    field(:account_id, :integer)
    field(:name, :string)
    field(:email, :string)
    field(:include, {:array, :string})
    embeds_one(:license, License)
  end

  def validate(attrs) do
    changeset(%__MODULE__{}, attrs)
  end

  def changeset(struct, attrs) do
    struct
    |> cast(attrs, [:name, :email, :account_id, :include])
    |> cast_embed(:license)
    |> validate_required([:name, :email, :account_id])
    |> validate_number(:account_id, greater_than: 0, less_than_or_equal_to: @max_int32)
    |> validate_include()
  end

  def validate_include(changeset) do
    changeset
    |> get_field(:include)
    |> validate_include_value(changeset)
  end

  defp validate_include_value(nil, changeset), do: changeset

  defp validate_include_value(includes, changeset) when is_list(includes) do
    if Enum.all?(includes, &(&1 in @allowed_includes)) do
      changeset
    else
      add_error(changeset, :include, "#{@invalid_include_value} #{@allowed_includes_str}")
    end
  end

  defp validate_include_value(_includes, changeset) do
    add_error(changeset, :include, @invalid_include_format)
  end
end

defimpl Jason.Encoder, for: ParamzWeb.Params.License do
  def encode(value, opts) do
    value
    |> Map.take([[:id, :type]])
    |> Jason.Encode.map(opts)
  end
end

defimpl Jason.Encoder, for: ParamzWeb.Params.Complex do
  def encode(value, opts) do
    current_value = Map.take(value, [:account_id, :name, :email, :include])

    if value.license do
      Map.put(current_value, :license, Map.take(value.license, [:id, :type]))
    else
      current_value
    end
    |> Jason.Encode.map(opts)
  end
end
```

## Changeset Errors

Here’s some simple code to convert changeset errors to a String.

```
defmodule Paramz.Utils.LogHelp do
  @moduledoc """
  Utility functions related to logging errors
  """

  @doc """
  If our app code gets back an error tuple from Ecto where the second
  element is an Ecto.Changeset this function can convert that value
  into a String for logging
  """
  @spec ecto_error(Ecto.Changeset.t()) :: String.t()
  def ecto_error(%Ecto.Changeset{} = changeset) do
    Ecto.Changeset.traverse_errors(changeset, fn {msg, opts} ->
      Enum.reduce(opts, msg, fn {key, value}, acc ->
        String.replace(acc, "%{#{key}}", _to_string(value))
      end)
    end)
    |> Enum.reduce("", fn {k, v}, acc ->
      joined_errors = Enum.join(v, "; ")
      "#{acc} #{k}: #{joined_errors}"
    end)
  end

  def ecto_error(changeset) do
    raise ArgumentError, "unsupported argument: #{inspect(changeset)}"
  end

  defp _to_string(val) when is_list(val) do
    Enum.join(val, ",")
  end

  defp _to_string(val), do: to_string(val)
end
```

## Testing

### Simple Endpoint - Success

#### Client side

```
$ curl -s http://localhost:4000/api/1234/90023377/simple

{"account_id":1234,"user_id":90023377}%
```

#### Server side

```
[info] GET /api/1234/90023377/simple
[debug] Processing with ParamzWeb.ParamCheckController.simple/2
  Parameters: %{"account_id" => "1234", "user_id" => "90023377"}
  Pipelines: [:api]
after Params call: %ParamzWeb.Params.Simple{account_id: 1234, user_id: 90023377}
[info] Sent 200 in 1ms
```

### Simple Endpoint - Bad Parameters, Failure

#### Client side

```
curl -s http://localhost:4000/api/1234/90abcdE77/simple

{"error":"Invalid parameter input.  user_id: is invalid"}%
```

#### Server side

```
[info] GET /api/1234/90abcdE77/simple
[debug] Processing with ParamzWeb.ParamCheckController.simple/2
  Parameters: %{"account_id" => "1234", "user_id" => "90abcdE77"}
  Pipelines: [:api]
[info] Sent 400 in 1ms
```

### Complex Endpoint - Success

#### Client Side

```
$ curl -s -H "Content-Type: application/json" -X POST -d '{
"name": "testing",
"email": "testing@google.com",
"license": {
  "id": "95055",
  "type": "license_reference"
}
}' http://localhost:4000/api/1234/complex\?include\[\]\=posts

{"account_id":1234,"email":"testing@google.com","include":["teams"],"license":{"id":95055,"type":"license_reference"},"name":"testing"}%
```

#### Server side logging

```
[info] POST /api/1234/complex
debug] Processing with ParamzWeb.ParamCheckController.complex/2
Parameters: %{"account_id" => "1234", "email" => "testing@google.com", "include" => ["posts"], "license" => %{"id" => "95055", "type" => "license_reference"}, "name" => "testing"}
Pipelines: [:api]
after Params call: %ParamzWeb.Params.Complex{
account_id: 1234,
name: "testing",
email: "testing@google.com",
include: ["posts"],
license: %ParamzWeb.Params.License{id: 95055, type: "license_reference"}
}
[info] Sent 200 in 2ms
```

### Complex Endpoint - Bad Parameters, Failure

#### Client Side

```
curl -s -H "Content-Type: application/json" -X POST -d '{
"name": "testing",
"email": "testing@google.com",
"license": {
  "type": "license_reference"
}
}' http://localhost:4000/api/1234/complex\?include\[\]\=posts

{"error": "Invalid parameter input. license: %{id: [\"can't be blank\"]}"}
```

#### Server Side

```
iex(1)> [info] POST /api/1234/complex
changeset errors: #Ecto.Changeset<
  action: nil,
  changes: %{
    account_id: 1234,
    email: "testing@google.com",
    include: ["posts"],
    license: #Ecto.Changeset<
      action: :insert,
      changes: %{type: "license_reference"},
      errors: [id: {"can't be blank", [validation: :required]}],
      data: #ParamzWeb.Params.License<>,
      valid?: false
    >,
    name: "testing"
  },
  errors: [],
  data: #ParamzWeb.Params.Complex<>,
  valid?: false
>
[debug] Processing with ParamzWeb.ParamCheckController.complex/2
  Parameters: %{"account_id" => "1234", "email" => "testing@google.com", "include" => ["posts"], "license" => %{"type" => "license_reference"}, "name" => "testing"}
  Pipelines: [:api]
[info] Sent 400 in 2ms
```

## The params library

There is a [params library](https://hex.pm/packages/params) that wraps Ecto
validation. It attempts to reduce boiler-plate code. However, that comes at a
cost in giving up control you might need. In addition, there is no active
maintainer for the library at the moment. I’d recommend not using this library
at the moment.
