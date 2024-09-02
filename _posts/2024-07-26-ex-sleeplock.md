---
layout: post
title: The ex_sleeplock library
date: 2024-06-27 12:41:00
description: Limiting concurrent processes in Elixir
categories: elixir
tags: kafka concurrent
giscus_comments: true
---

I finally got around to publishing the [ex_sleeplock](https://hex.pm/packages/ex_sleeplock)
library to hex.pm. This is a useful library for a particular problem.

## Reason for Library

The reason this library got developed was because I was dealing with multiple
services that served an API and consumed Kafka messages. Both of these activities
required a database connection. Database connections available in limited quantities
(set by the pool size). The API should have priority if a decision has to made on handing
out a connection. The API should always have an available connection to process a request.

It's clear that a service can have lots of simultaneous requests that it needs
to handle. For the services I was working with the service handled many simultaneous
requests well. The problem was processing messages from Kafka.

The service used the [brod](https://hexdocs.pm/brod/readme.html) library. In particular,
it used the [brod_group_subscriber_v2](https://hexdocs.pm/brod/brod_group_subscriber_v2.html)
module to consume messages. brod_group_subscriber_v2 starts a process for each partition that
is assigned to this client node.

The messages can arrive in parallel. Assume a topic with 64 partitions. The Kafka
clients act as a consumer group. So Kafka splits the partitions between each member
of that consumer group. Assuming we have 4 clients, for example, it means that
16 messages could arrive "simultaneously".

## How Brod Passes Messages to An App

Brod provides a behaviour definition that the app must implement. The main callback function
is `handle_message/2`. This is defined in the brod Erlang code as:

```
-callback handle_message(brod:message(), State) ->
      {ok, commit, State}
    | {ok, ack, State}
    | {ok, State}.
```

When a message arrives for any of brod partition processes the brod code calls the app's
implementation of `handle_message/2`. The app is responsible for processing the
message and returning a value that lets brod know whether to "commit" the offset of
the read message, just "ack" the message or tell Kafka nothing.

The callback is typically code that is going to examine the incoming Kafka message
and validate it. The code might convert it into something that the app can more
easily process. In any case, at some point a processing function is called that
is what needs to talk to the database. The processing function is something like
this:

```
def process_incoming_message(message) do
   <message processing logic>
end
```

## Database Pool Size

The pool size was typically set to 10 for each pod. With 16 partitions assigned it
is easy to hit a situation where the Kafka processing used up all 10 of the connections.
And if there is a huge flood of messages this situation might go on for a while.
The API could have trouble getting a database connection before a timeout is reached.

## Separate the Repos?

One possible solution is to separate the Ecto Repos. Declare one as the WebRepo
and give it the number of connections you think it might want. Declare another
as the KafkaRepo and give that Repo its own connections. This works since now
the API is using its own connections. However, its also a bit awkward given
that there are times when no messages are coming in from Kafka. No connections
are actually needed for it. But the API might be getting hit harder than usual.
Having the 10 total connections available would be good.

## Solving the Problem with ex_sleeplock

To solve this the ex_sleeplock library was created. What this does is create a
named process with an application specified level of concurrency. The message
processing code did call this:

```
def process_incoming_message(message) do
   <message processing logic>
end
```

Instead it became this:

```
def process_incoming_message_throttled(message) do
  ExSleeplock.execute(:kafka_consumer_throttle, fn -> process_incoming_message(message) end)
end
```

The Kafka message processor started calling `process_incoming_message_throttled/1`
instead of `process_incoming_message/1`. All the other code remained the same. The
end result was the number of processes that could call `process_incoming_message/1`
was limited to the level of concurrency specified in an app config file.

## Using the Library

The library is fairly easy to use. You can even configure the locks that you want in
our application config and the library will create the processes for each lock when
it starts up. For example:

```
config :ex_sleeplock, locks: [%{name: :kafka_consumer_throttle, num_slots: 2}]
```

This configures the library to allow a maximum of two processes to be in the
`process_incoming_message/1` code at once.

The library supervises all the lock processes. So there isn't extra things to think
about or maintain in the code. Which seemed like a good thing as well.

## So That's The Story

Anyway, I'm happy to share this library. Its implementation was based off an existing
Erlang library called [sleeplocks](https://hex.pm/packages/sleeplocks). We used this
at first but we wanted to have the library itself manage the supervision and we
wanted some additional things like telemetry events and creation of the locks via
a config file. That library is perfectly fine though. And it solves the same problem.

If you use the library and want to have it do something else, do things differently
or whatever then I'm happy to review pull requests. The repo for the project
is [ex_sleeplock library](https://github.com/fmcgeough/ex_sleeplock).
