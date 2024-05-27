---
layout: page
permalink: /teaching/
title: teaching
description: Helping people learn makes me happy
nav: true
nav_order: 6
---

I spend time trying to develop Elixir code that can help developers
learn the language and how best to apply it. There are those that
think the time is better spent developing libraries. I understand
the thought process but I think ultimately having someone understand,
rather than just use, Elixir code is very worthwhile.

- [brod_mimic](https://github.com/fmcgeough/brod_mimic) - the main means of interacting
  with Kafka for most Elixir developers is the brod library. This library is written in
  Erlang. It can be a daunting task for developers new to Elixir to also take on
  learning Erlang as well. This project was developed to do a "port" of brod to Elixir.
  This allows developers to see how things are expressed in Erlang vs Elixir.

- [Elixir Supervision](https://github.com/fmcgeough/elixir-supervision). One of the more
  interesting aspects of Elixir (Erlang) is the actor model implementation and the
  principle of "let it crash". There are simple but powerful constructs in the language
  that allow a developer to easily build a supervision tree so that processes are
  restarted if (when) they crash. But most Elixir developers struggle with understanding
  the mechanics. This repo is sample Elixir code meant to demonstrate some aspects of
  Elixir Supervision.
