---
layout: post
title: Elixir, Mongo, Exploring Library Code
date: 2017-03-24 08:53:13
description: Accessing MongoDB with Elixir
categories: Elixir
tags:
---

Since I'm relatively new to Elixir one of the ways that I learn more is by looking
at the source code for dependencies that get brought in. I needed to use Mongo for
a current task and I added a dependency on [mongodb](https://hex.pm/packages/mongodb).
This blog post goes through some initial exploration of how the mongodb library is
implemented and some interesting things about Elixir.

The first thing that I do when looking at a dependency is to examine what it is
dependent upon. In the case of mongodb I see :

```
defp deps do
  [{:connection,    "~> 1.0"},
   {:db_connection, "~> 1.0"},
   {:ex_doc,        ">= 0.0.0", only: :dev},
   {:earmark,       ">= 0.0.0", only: :dev}]
end
```

ExDoc is a documentation generation tool for Elixir and Earmark is an Elixir Markdown
converter. Both are marked as dev only so lets ignore those for now. The first thing
that interested me was [db_connection](https://hex.pm/packages/db_connection).
This is because I know we use DBConnection in the
mongo code when we start up a connection to Mongo using something like :

```
iex(1)> {:ok, _} = Mongo.start_link(database: "test", name: :mongo)
```

Examining the Mongo module (mongo.ex) in the mongodb library we see :

```
def start_link(opts) do
  DBConnection.start_link(Mongo.Protocol, opts)
end
```

DBConnection is its own module. Here's the current DBConnection.start_link code :

```
def start_link(conn_mod, opts) do
  pool_mod = Keyword.get(opts, :pool, DBConnection.Connection)
  apply(pool_mod, :start_link, [conn_mod, opts])
end
```

That first line is looking in the options supplied by the caller to see if they
supplied a :pool. If they didn't then we get DBConnection.Connection module
as the pool. The comments note that this is a pool of 1.

The second line - [apply](https://hexdocs.pm/elixir/Kernel.html#apply/2) - is a way of
dynamically calling functions. The same facility is available in Erlang. Its found
in the Elixir Kernel module. The first parameter is the module. The second parameter is
the function within the module. The third parameter is the arguments to the function.
In the case of the default pool (DBConnection) the code is calling DBConnection start_link.
But how does this work? If you look in DBConnection.Connection (connection.ex) you
see the following defined function :

```
def start_link(mod, opts) do
  start_link(mod, opts, :connection)
end
```

So, from our start_link call, conn_mod becomes mod in the argument and any option parameters
are the second parameter. Remember our conn_mod is Mongo.Protocol. Lets go back to the
mongodb library to find that. defmodule Mongo.Protocol is in protocol.ex. It has the macro
use DBConnection right after the declaration of the defmodule. This will invoke the **using**
function in DBConnection when it expands the macro. Looking back at DBConnection you can
see that this is doing the following :

```
@behaviour DBConnection
```

So, its defining something called a [behavior](https://hexdocs.pm/elixir/1.4.5/behaviours.html).
As noted in the doc, behaviors provide a way to :

- define a set of functions that have to be implemented by a module;
- ensure that a module implements all the functions in that set.

What DBConnection does (which is not that uncommon) is that it actual defines functionality
for all of the functions that someone implementing a DBConnection can implement. The
implementations consist of "not implemented" type messages. Then DBConnection enumerates all
the functions that can be overridden by the impelementor. You can find the functions
defined as overrideable by searching for the string "defoverridable". The following
functions are defined as allowing overrides :

```
- connect: 1
- disconnect: 2
- checkout: 1
- checkin: 1,
- ping: 1
- handle_begin: 2
- handle_commit: 2
- handle_rollback: 2
- handle_prepare: 3
- handle_execute: 4
- handle_close: 3
- handle_declare: 4
- handle_first: 4
- handle_next: 4
- handle_deallocate: 4
- handle_info: 2
```

Why does DBConnection do this? The idea is that the set of functions defined are all the
ones that a "database type thing" might implement but that some "database type things" won't
implement all of them. For example, Mongo doesn't have transactional control. The document you
store in Mongo ends up being kind of the equivalent of a transaction in a relational database.
So, the Mongo Protocol is probably not going to implement begin, commit, rollback. Lets go
look at the Mongo.Protocol code to see what functions it does implement. The easiest way to
find these is to see what is publicly available via a def function (the defp functions are private
parts of the protocol's implementation). Here's what I found :

```
- connect: 1
- checkout: 1
- checkin: 1
- handle_execute: 4
- handle_info: 2
- ping: 1
```

That looks like it. Although there is a `handle_execute_close: 4` function defined as well. This
doesn't appear to be used.

The Mongo.Protocol doesn't have any moduledoc nor doc on any of the functions within it. This is
something that needs contribution. Contrast it with the DBConnection module which has extensive
documentation on both the module and function level. Of course, DBConnection was defining a
behavior and it stands to reason that it needed to be fairly well documented to be accepted
as a proper behavior that various folks could implement for their own project.

In any case with just with a bit of exploring you can see a lot of pieces of how Elixir libraries
are structured and how developers that came before you tried to extract common functionality into
protocols that others could then implement.

Next time I'll delve into more details on the mongodb library.
