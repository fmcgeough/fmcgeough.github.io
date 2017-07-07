---
layout: post
title: 'Elixir, Phoenix and Default Logging'
categories:
- Elixir
tags: []
---
If you are using the default Logger in your Elixir/Phoenix web app
with no further configuration then your logs will rotate rather quickly.
If you are using Distillery then, by default, your log files will be in
var/log and they'll be named like:

```
erlang.log.1
erlang.log.2
erlang.log.3
erlang.log.4
erlang.log.5
```

and they won't exceed 100,000 bytes. This isn't very much in a production deployment
for a web app so you probably want to increase this so that you can tail a single
log file to get a quick indication of what is going on with your app.


The default logger is subject to the [Erlang environment variables](http://erlang.org/doc/man/run_erl.html#environment_variables).
The one you'll want to modify to increase the size of your logging file is: RUN_ERL_LOG_MAXSIZE. From
the documentation:

```
RUN_ERL_LOG_MAXSIZE
  The size, in bytes, of a log file before switching to a new
  log file. Defaults to 100000, minimum is 1000, maximum is
  about 2^30.
```
