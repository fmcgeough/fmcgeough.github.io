---
layout: post
title: Elixir, Reading Dynamo Streams (Part 2)
date: 2019-10-22 08:53:13
description: How to read from Dynamo stream in Elixir
categories: Elixir
tags: Elixir
---

Just made a small change to this project today since I didn't have a lot of
time to work on it and got involved in looking at Clojure. The project now
has some of the standard dependencies you regularly see in "real" Elixir
project: excoveralls, dialyxer, credo, ex_doc. And I've refactored table.ex
to be more general purpose and moved the creation of the actual test table for
this project to user_activity.ex.

There is a couple of unit tests now just to show an approach to mocking requests
with ExAws. Its obviously not necessary for this test project but it seemed like
something someone coming to Elixir and stumbling on these posts might like to see.
In any case, everything is available at [dynamo_streamer](https://github.com/fmcgeough/dynamo_streamer) and I'm tagging the work I'm doing. This particular increment
is v0.3. :-)

Hopefully tomorrow there will be records into DynamoDB and the beginning of
dealing with streams, shards and iterators.
