---
layout: post
title: Elixir, Reading Dynamo Streams and Layoffs
date: 2019-10-20 08:53:13
description: How to read from Dynamo stream in Elixir
categories: Elixir
tags: Elixir
---

I was working remotely for Weedmaps up until recently writing Elixir with a group of really great
coworkers. Unfortunately, Weedmaps laid off 25% of its workforce and I was let go, along with some
other really great productive people - if you need devs, qa, project managers please ask me. I know
a bunch that I really enjoyed working with. I figured while I search for a new position I'd use some
of the time to blog a bit about Elixir. This will be a bit rambling since I'm writing in-between
phone calls and interviews.

Elixir is a relatively young language but has a great community of developers that contribute
some very useful code. One of the reliable libraries that I've used multiple times is: [ex_aws](https://github.com/ex-aws/ex_aws).

This library provides base services for AWS API's. If you've used Python for AWS via the Boto library you
can think of ex_aws as providing similar functionality. However, the approach the library developer took was
to separate the base functionality (XML and JSON request building, exponential backoff) from the service
libraries that provide an actual interface to an API service. I really like this approach for wrappers around
the AWS APIs because I don't want to have code for every single service that AWS provides in my app. Being
able to just setup dependencies I need means that I probably will be able to look at every piece of code
and evaluate any security problems or issues that might impact upgrades in the future.

## DynamoDB

AWS provides a key-value store called DynamoDB. Its got several nice features that
make it a popular choice for companies using AWS. Its self-managed and can scale based
on usage patterns, for example. One of its other nice features is that there is a stream
API that gives you access to changes that have been made over the last 24 hours. For
this blog post I'll build a little github repo that demonstrates that stream API.

Another nice feature of DynamoDB is that you can use docker to run a local copy of DynamoDB.
I'm a big fan of this. If only all the AWS services provided some means of development and
testing without using AWS itself. In any case, it'll allow you to download the repo I build
in github and try it out without even having an AWS account. Here's the docker commands you
can use to get a local DynamoDB running:

```
docker pull amazon/dynamodb-local
docker run -p 8000:8000 amazon/dynamodb-local
```

There are two separate libraries that plug into ex_aws for DynamoDB: 1) ex_aws_dynamo that covers the API
to access to your table and 2) ex_aws_dynamo_streams that covers the separate API for the streams functionality.
Since I want to demonstrate how to use the stream API I'll need the base API in order to create a table that
provides stream data and insert records that show up in the stream. The project is in github at:
`https://github.com/fmcgeough/dynamo_streamer`

## Creating a Simple Elixir project

Mix is the ubiquitous command line tool for Elixir. I've talked to a number of Rails developers that liked this
approach (as opposed to bundle, rails, rake, etc). In any case if you're new to Elixir it'd be good to spend
some time becoming familiar with all that mix offers. Here's what I used to create a new bare-bones Elixir project.

```
mix new dynamo_streamer
* creating README.md
* creating .formatter.exs
* creating .gitignore
* creating mix.exs
* creating config
* creating config/config.exs
* creating lib
* creating lib/dynamo_streamer.ex
* creating test
* creating test/test_helper.exs
* creating test/dynamo_streamer_test.exs

Your Mix project was created successfully.
You can use "mix" to compile it, test it, and more:

  cd dynamo_streamer
  mix test

Run "mix help" for more commands.
```

The main configuration file for every Elixir project is the mix.exs file. This is where I'll add the
dependencies for the 2 dynamo libraries. You can learn the syntax needed for a particular library by
visiting [hex.pm](https://hex.pm/) which is the package manager for the Erlang ecosystem.

We'll add the 2 dynamo libraries as dependencies in mix.exs (they are both dependent upon ex_aws).
Open up the config file in the deps/ex_aws_dynamo_streams/config/config.exs and you'll see the config
that is necessary in order to use the local DynamoDB. Trim down the README to something really basic
and I'll push that all to github at: `https://github.com/fmcgeough/dynamo_streamer`. I'll add tags
along the way and do pull requests so it'll hopefully be easy to follow the steps if you want to.

## But the Libraries Won't Help you Understand

The thing about the wrapper libraries that plug into ex_aws is that for the most part they have
very bare-bones documentation (sometimes none). In any case, when you are using a new AWS service
via Elixir you'll need to open up the AWS doc page.

- [DynamoDB API Doc](https://docs.aws.amazon.com/en_pv/amazondynamodb/latest/APIReference/API_Operations_Amazon_DynamoDB.html)
- [DynamoDBStreams API Doc](https://docs.aws.amazon.com/en_pv/amazondynamodb/latest/APIReference/API_Operations_Amazon_DynamoDB_Streams.html)

## Creating a DynamoDB Table

OK, so in order to use DynamoDB Streams we need a table created that enables streams. I'll create a
new local branch:

```
git checkout -b create_table
```

Some bookkeeping first. Lets delete the files : lib/dynamo_streamer.ex and test/dynamo_streamer_test.exs.
Those were generated as templates by the mix new and we won't need them. Now I'll create a file called
lib/table.ex to hold the code related to creating and setting up our table.

When you decide to use an Elixir library you need to examine its doc as well.

- [ex_aws_dynamo Doc](https://hexdocs.pm/ex_aws_dynamo/ExAws.Dynamo.html)
- [ex_aws_dynamo_streams Doc](https://hexdocs.pm/ex_aws_dynamo_streams/ExAws.DynamoStreams.html)

Another issue. If you try to use just these libraries by themselves it won't work. You'll need
two other libraries: hackney and poison. Hackney is used to send the web requests and poison is
used to decode responses.

OK, so that's sorted. What kind of table do I want to create in Dynamo? Well for this little
demo I'll create a table called "user-activities" that tracks actions by users in our UI over the
last hour. I'll remove the rows from the table after one hour using Dynamo's TTL. These removed
rows show up in the stream so it'll allow me to write code that differentiates between rows that
were inserted and rows that were removed by the TTL.

When you create a Dynamo table you'll have to decide on its partition key as well. Since we're only
interested in using this as a demonstration on how to read from DynamoDB Streams we'll use a generated
id. This partition key is defined as a HASH key. You can also specify a sort key. We'll use email
as the sort key. The sort key is defined as a RANGE key. You can only have one of each for your
DynamoDB table. The function ends up looking like this:

```
  def create_table(tablename) do
    tablename
    |> Dynamo.create_table(
      [id: :hash, email: :range],
      [id: :string, email: :string],
      1,
      1
    )
    |> ExAws.request()
  end
```

For the table definition itself we can define a module with a struct that ex_aws_dynamo will encode.
For the id generation, I pulled in the [puid library](https://hexdocs.pm/puid/Puid.html).

Our simple table definition is simply:

```
defmodule DynamoStreamer.UserActivity do
  @derive ExAws.Dynamo.Encodable

  alias DynamoStreamer.Id

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

## Adding Supervision and Wrapping up

Finally, when the project was generated we didn't use the `--sup` option to get a supervisor. I
feel like I'll need it to generate data and read from the stream independently. Its easy enough to
add after the fact. I'll modify the applications section of the mix.exs so that we start a
DynamoStreamer.Application on startup. You can read about that by using `mix help compile.app`.

So, that gives us some of the base things in place. Next stop is to start generating some data to
populate our table and stream. I'll hopefully tackle that tomorrow.
