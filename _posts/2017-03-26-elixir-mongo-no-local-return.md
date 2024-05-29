---
layout: post
title: Elixir, Mongo, No Local Return, Oh My!
date: 2017-03-26 08:53:13
description: Accessing MongoDB with Elixir
categories: Elixir
tags: Elixir
---

[Mongodb](https://hex.pm/packages/mongodb) library uses
the [dbconnection](https://hex.pm/packages/db_connection) library,
which in turn uses the [connection](https://hex.pm/packages/connection) library. Both
db_connection and connection libraries use a technique that may appear puzzling at first. Here's an example of the
code I'm talking about from defmodule DBConnection :

```
def handle_begin(_, state) do
  message = "handle_begin/2 not implemented"
  case :erlang.phash2(1, 1) do
    0 -> raise message
    1 -> {:error, RuntimeError.exception(message), state}
  end
end
```

So, its using an Erlang method called phash2 in a case statement and then handles
the result of either 0 or 1. So what is phash2? The [Erlang doc](http://erlang.org/doc/man/erlang.html#phash2-2) explains.

> erlang:phash2(Term, Range) -> Hash Portable hash function that gives the same hash for the same Erlang term regardless of machine architecture and ERTS version (the BIF was introduced in ERTS 5.2). The function returns a hash value for Term within the range 0..Range-1. The maximum value for Range is 2^32. When without argument Range, a value in the range 0..2^27-1 is returned.

The code is passing in 1 as the Range. The hash value will be in the range 0..0? So this will always return 0. What gives?
The answer can be seen in the first use of this technique in DBConnection.connect. It says :

> # We do this to trick dialyzer to not complain about non-local returns.

So, yes. The call into the Erlang hash function will always return 0. But if
the code just had "raise message" then dialyzer would not be happy since at that
point there would be no possibility that the function could ever return a value and
that's a dialyzer no-no. Since the library developers wanted dialyzer to run cleanly
they decided to use this little trick and then provided a bit of documentation in
the "unused" return value. This function should never be called. It is supposed to
implemented by the protocol for the particular datasource you are using (Postgresql,
Mongo, or whatever).

You can read more about Elixir, raise, try and rescue in the [Elixir documentation](http://elixir-lang.org/getting-started/try-catch-and-rescue.html#errors).
