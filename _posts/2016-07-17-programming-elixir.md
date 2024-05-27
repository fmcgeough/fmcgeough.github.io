---
layout: post
title: Programming Elixir
date: 2016-07-17 08:53:13
description: Learning Elixir
categories: Elixir
tags: Elixir
---

For learning Elixir I started with the ubiquitous Dave Thomas and "Programming
Elixir". I'm bouncing back and forth between reading through that book and
watching the ConFreaks videos of the last two year's Elixir conferences (and
thinking about whether I want to pay to go to this year's conference which is
in Orlando).

"Programming Elixir" has been pretty good. The exercises at the end of the
chapter cover the concepts discussed. I was stumped for a couple of minutes
looking at "define a function head with the defaults". But after working through
the example it made sense and resulted in code that seemed to make sense to
me.

The thing is in Elixir (and Erlang) the function head (declaration) is a
separate entity from the function body. Which is an odd thing to wrap your
head around if you come from almost any other language background. In the
case of the example with default parameters in the book you can do this :

```
    defmodule DefaultParams1 do
      def func(p1, p2 \\ 123)

      def func(p1, 99) do
        IO.puts "you said 99"
      end

      def func(p1, p2) do
        IO.inspect [p1, p2]
      end
    end
```

If you call DefaultParams1.func(1) then the match will be on that first func without
a body. The second parameter is added as a default and the search for a match
continues. Obviously a contrived example for the book but I do like the approach of
separating the definition of what we might want to supply as defaults from the rest
of the code.

Just beginning the journey into Elixir but its very interesting so far.
