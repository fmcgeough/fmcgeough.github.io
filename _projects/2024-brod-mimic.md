---
layout: page
title: brod-mimic
description: Elixir code exploration of the Erlang brod Kafka library
img:
importance: 1
category: educational
related_publications: false
---

An Elixir project to explore the Erlang
library [brod](https://github.com/kafka4beam/brod/tree/master) by porting
it to Elixir. It resides in the github
repo [brod_mimic](https://github.com/fmcgeough/brod_mimic).

This project was created for a couple of reasons.

- The brod library is the most popular means in Elixir to interface with Kafka.
- Elixir developers generally lack knowledge on how to read the Erlang code.
  The brod code is complex and the end result is that Elixir developers may
  try and use the library as a black-box and that falls apart if problems occur
  that require some knowledge of how the library is talking to Kafka.
- I figured having an Elixir implementation would allow Elixir developers to
  examine this code and then have a clearer understanding of how brod works.
  To make this a reality I needed to:
  - convert the brod code file by file (so that a developer looking at the
    code could align what is in this library with the brod code). So, for
    example, in the brod library there is a src/brod_consumers_sup.erl file.
    In this library there is a lib/brod_mimic/consumers_sup.ex file.
  - modify doc so that the information aligns with usage from Elixir (the
    brod documentation is written from an Erlang perspective)
  - add documentation that clarifies some of the more complicated aspects of
    the code

Developers should not use this library in lieu of brod. It's not nearly production
ready and the brod library has been used for years by all sorts of companies. This
project is meant to be educational.
