---
layout: post
title: Elixir and Ecto's telemetry events
date: 2024-09-02 12:00:00
description: How to use Ecto's telemetry events
categories: elixir
tags: telemetry
giscus_comments: true
---

I created a simple sample Elixir project to demonstrate how to use Ecto's telemetry
events. It's available in [github](https://github.com/fmcgeough/demo_telemetry).

There is one telemetry event that is ordinarily of interest from Ecto. It is generated
by the [ecto_sql](https://hex.pm/packages/ecto_sql) library in the
`ecto_sql/lib/ecto/adapters/sql.ex` file. This event is fired for every database
activity (select, insert, update, delete, begin, rollback, commit, etc).

The project shows how to:

- use telemetry_prefix on your repo to change the name of the event
- use telemetry_prefix with your queries to allow you to easily identify what
  database activity has occurred in your metrics handler
- what event name is if you do not use telemetry_prefix for your repo.
