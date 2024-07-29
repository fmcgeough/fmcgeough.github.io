---
layout: page
title: Elixir Sleeplock
description: Allow only a certain number of processes to execute a block of code at one time
img:
importance: 1
category: published
related_publications: false
---

Published to hex.pm [ex_sleeplock](https://hex.pm/packages/ex_sleeplock).

An easy approach to throttling the number of processes allowed to execute a block of code simultaneously in Elixir.

Started from the Erlang project https://hex.pm/packages/sleeplocks. Rewrote in Elixir with added monitoring of processes that take locks and dynamic supervision of the locks themselves. Thanks to Isaac Whitfield who created the sleeplocks library.

This library provides an app with the ability to use sleep locks. So what are sleep locks?

Sleep locks allow an app to throttle the number of processes that are allowed to be executing some section of code at any one time. It does this by creating the number of slots that the caller requested and only allowing that number of processes to get the lock at any one time.
