---
layout: post
title: Elixir, Reading Dynamo Streams (Part 3)
date: 2019-10-23 08:53:13
description: How to read from Dynamo stream in Elixir
categories: Elixir
tags: Elixir
---

Finishing up this small project to read a DynamoDB stream with Elixir. The
github repo for the project is: [dynamo_streamer](https://github.com/fmcgeough/dynamo_streamer). This post will cover generating random data to insert into the
table and reading from the stream associated with the table.

## Writing Data

The module that represents the table data is simple (since the main purpose is
just to show how to use the stream) :

```
defmodule DynamoStreamer.UserActivity do
  @derive ExAws.Dynamo.Encodable

  alias DynamoStreamer.{Id, Table}

  defstruct [:id, :email, :activity, :ttl]

  def new(email, activity) do
    %__MODULE__{
      id: Id.generate(),
      ttl: DateTime.utc_now() |> DateTime.add(3_600, :second) |> DateTime.to_unix(),
      email: email,
      activity: activity
    }
  end
end
```

This uses the @derive attribute. You can read more about this in the Elixir doc at
[Protocols](https://elixir-lang.org/getting-started/protocols.html). The bottom line
is that your struct will be converted into a map with decorations on it indicating the type. For example, lets suppose that I have a UserActivity struct that looks
like this:

```
%DynamoStreamer.UserActivity{
  activity: %{page: "bGqh9Qqhhg29r3PdLtRp", task: "buying"},
  email: "PTnFGpDrQ8h66j9nff8g@gmail.com",
  id: "jLn22NHrRHBM9BfjgNtm",
  ttl: 1571842348
}
```

When its encoded by the ex_aws_dynamo library for transmission to DynamoDB API it'll look like this:

```
%{
  "activity" => %{
    "M" => %{
      "page" => %{"S" => "bGqh9Qqhhg29r3PdLtRp"},
      "task" => %{"S" => "buying"}
    }
  },
  "email" => %{"S" => "PTnFGpDrQ8h66j9nff8g@gmail.com"},
  "id" => %{"S" => "jLn22NHrRHBM9BfjgNtm"},
  "ttl" => %{"N" => "1571842348"}
}
```

So, its still a map with your keys (as strings) at the top-most layer but the values now have those letters
as keys in front of them. You can look thru the DynamoDB doc but in the example above: "M" = map, "S" = string,
and "N" = number.

An important thing to note is that data comes back to you from DynamoDB like that too. But I'm getting ahead
of myself a bit. First, I'll generate a random function in user_activity to generate some fake data and insert
a few records into the table and make sure they are there.

```
  def random do
    new("#{Id.generate()}@gmail.com", %{
      page: Id.generate(),
      task: Enum.random(["buying", "selling", "viewing", "data_entry"])
    })
  end
```

Now insert a few records:

```
iex> alias DynamoStreamer.UserActivity
iex> alias ExAws.Dynamo
iex> 1..4 |>
   Enum.each(fn _ -> Dynamo.put_item("user-activities", UserActivity.random()) |>
   ExAws.request()
   end)
```

Now calling Dynamo.scan should return the 4 records

```
iex> Dynamo.scan("user-activities") |> ExAws.request()
{:ok,
 %{
   "Count" => 4,
   "Items" => [
     %{
       "activity" => %{
         "M" => %{
           "page" => %{"S" => "hTg3Tt77mrPJ3MqBFTgp"},
           "task" => %{"S" => "buying"}
         }
       }
     },
     etc..
```

As you can see the data is there in the "Items" value but its got all the DynamoDB signifiers in
front on it and nobody would want to work with it that way. You can take the data and make it into
a regular map in this way:

```
{:ok, %{"Items" => items}} = ExAws.Dynamo.scan("user-activities") |> ExAws.request()
items |> Enum.map(&ExAws.Dynamo.Decoder.decode(&1, as: DynamoStreamer.UserActivity))
```

This yields values that look like this:

```
%DynamoStreamer.UserActivity{
    activity: %{"page" => "D7rJ8gjH26N98g7JJgjF", "task" => "data_entry"},
    email: "4pDmHmj9GhpjntT3m8qR@gmail.com",
    id: "fpqhngnn64ftdN3pjDgg",
    ttl: 1571843947
  }
```

So, the `activity` is not decoded to atoms because its a map. If you wanted to do that you'd have to
implement your own decode which will be called after the built in decoder is called.

OK, so we've got records. Do we have streams? Well calling Dynamo.describe_table("user-activities")
gives back a bunch of interesting info including "LatestStreamArn". There's some other interesting
info in there so lets make a struct that encapsulates that.

```
defmodule DynamoStreamer.StreamInfo do
  defstruct [:stream_arn, :stream_label, :enabled, :view_type]

  @type t :: %__MODULE__{
          stream_arn: binary | nil,
          stream_label: binary | nil,
          enabled: boolean | nil,
          view_type: binary | nil
        }

  def new(%{
        "Table" => %{
          "LatestStreamArn" => stream_arn,
          "LatestStreamLabel" => stream_label,
          "StreamSpecification" => %{
            "StreamEnabled" => enabled,
            "StreamViewType" => view_type
          }
        }
      }) do
    %__MODULE__{
      stream_arn: stream_arn,
      stream_label: stream_label,
      enabled: enabled,
      view_type: view_type
    }
  end

  def new(_), do: {:error, "Unexpected format"}
end
```

As it turns out, we need the stream_arn for a couple of things. We use it for listing shards
using the describe_stream function and we need it to get a shard iterator which we need to
get the records themselves.

I've put together all the pieces and pushed it up to github. There are lots of other details
that you'd need to work out in order to use in a real application but hopefully it helps get
you started if you're just curious about how DynamoDB works with Elixir.
