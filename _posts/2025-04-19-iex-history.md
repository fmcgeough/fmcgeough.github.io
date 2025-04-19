---
layout: post
title: Elixir - Iex History
date: 2025-04-19
description: How to enable history for Elixir iex
tags:
categories: elixir
giscus_comments: true
---

There are any number of notes around showing how to enable history in iex (the Elixir language repl). You need to set a system environment variable for this to work. Add the following line:

```
export ERL_AFLAGS="-kernel shell_history enabled"
```

to either `~/.bash_profile` (for bash users) or `~/.zshev` (for zsh users).

You may need to completely exit iTerm if using OS X to enable iex history after setting up this environment variable.

This enables history in both iex and erl (Erlang repl). Note: the history is shared between iex and erl.
