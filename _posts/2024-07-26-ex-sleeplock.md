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

The reason this library got developed was because I was dealing with multiple
services that served an API and consumed Kafka messages. Both of these activities
required a database connection. And, not only are database connections available
in limited quantities (set by the pool size) but we wanted the API to always
have an available connection to process a request.

It's clear that a service can have lots of simultaneous requests that it needs
to handle. For the services I was working with the service handled these rather
well. The problem was processing messages from Kafka.

The service used the [brod](https://hexdocs.pm/brod/readme.html) library. In particular,
it used the [brod_group_subscriber_v2](https://hexdocs.pm/brod/brod_group_subscriber_v2.html)
module to consume messages. This module starts a process for each partition that
the allocation (pod) is assigned. The messages can arrive in parallel. In a system
with a 64 partition Kafka topic and 4 pods each pod would end processing 16 partitions.
That means 16 messages could arrive "simultaneously". There is some brod boilerplate
code to receive messages.

The pool size was typically set to 10 for each pod. So we could easily have a
situation where the Kafka processing used up all 10 of the connections.

One possible solution is to separate the Ecto Repos. Declare one as the WebRepo
and give it the number of connections you think it might want. Declare another
as the KafkaRepo and give that Repo its own connections. This works since now
the API is using its own connections. However, its also a bit awkward given
that there are times when no messages are coming in from Kafka. No connections
are actually needed for it. But the API might be getting hit harder than usual.
Having the 10 total connections available would be good.

To solve this the ex_sleeplock library was created. What this does is create a
named process with an application specified level of concurrency. The code that
was something like this:

Then the code would call a function like this:

```
def process_incoming_message(message) do
   <message processing logic>
end
```

became instead:

```
def process_incoming_message_throttled(message) do
  ExSleeplock.execute(:kafka_consumer_throttle, fn -> process_incoming_message(message) end)
end
```

The the Kafka consumer code was switched to call `process_incoming_message_throttled/1`
instead of `process_incoming_message/1` and all the code remained the same. The only
thing that was done was what we wanted. The number of processes that could call
`process_incoming_message/1` was limited to the level of concurrency specified by the
app.

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

Anyway, I'm happy to share this library. Its implementation was based off an existing
Erlang library called [sleeplocks](https://hex.pm/packages/sleeplocks). We used this
at first but we wanted to have the library itself manage the supervision and we
wanted some additional things like telemetry events and creation of the locks via
a config file. That library is perfectly fine though. And it solves the same problem.
