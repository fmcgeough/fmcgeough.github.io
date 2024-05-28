---
layout: post
title: Monitor Postgres pgbouncer
date: 2017-10-13 08:53:13
description: How to monitor the Postgresql load balancer pgbouncer
categories: postgresql
tags: postgresql
---

[Pgbouncer](https://pgbouncer.github.io/) is widely used with Postgres to provide connection pooling.
Its an easy-to-use and easy-to-install piece of software. The general idea is
to specify n number of connections allowed to pgbouncer and m connections
allowed to Postgres itself where m is much less than n. A typical configuration
is to set pgbouncer to transaction mode. This allows pgbouncer to multiplex
the "real" connections to Postgres as transactions are committed or
rolled back.

To connect to Postgres you point your app at the pgbouncer. You configure
the pgbouncer so that it knows when a connection is made to a database
it actually means that the app wants to talk to that database on the
server on which Postgres is running. pgbouncer implements the Postgres
wire protocol so that to the driver your app uses pgbouncer looks like
the Postgres database server.

pgbouncer also provides an "internal" database that you can connect to
called pgbouncer. Once you connect to the pgbouncer database you can
execute ["SHOW"](https://pgbouncer.github.io/usage.html) commands that
provide information on the current state of pgbouncer.

Note: By default the SHOW commands will include the header and use the
'|' character as a delimiter. You can override this behavior in psql using
the "-F" option and the "--no-align" option. The show commands do
not allow the use of criteria or joins. You can work around this if you
use a foreign data wrapper.

If you want to connect to the pgbouncer you use:

```
psql -p <your port> -h <pgbouncer host> -U postgres pgbouncer
```

or

```
psql -F ',' --no-align -p <your port> -h <pgbouncer host> -U postgres pgbouncer
```

if you want to specify a comma delimiter and basic data output.

Once connected with psql you should do:

```
 SHOW POOLS;
```

What does this command show? All of the databases that pgbouncer allows connections to are shown including a "pgbouncer" database that represents the bouncer itself. That row can be ignored. This is what the column data means:

- cl_active: Connections from clients which are associated with a PostgreSQL connection.
- cl_waiting: Connections from clients that are waiting for a PostgreSQL connection to service them.
- sv_active: Connections to PostgreSQL that are in use by a client connection.
- sv_idle: Connections to PostgreSQL that are idle, ready to service a new client connection.
- sv_used: PostgreSQL connections recently released from a client session.
- sv_tested: PostgreSQL connections in process of being tested.
- sv_login: PostgreSQL connections currently logging in.
- maxwait: The length of time the oldest waiting client has been waiting for a connection.

You want to ensure maxwait = 0. This means nobody is waiting for a connection.
You want your cl_active to be a fair distance from your configured default_pool_size.

You'll want to keep an eye on these stats for a bit if there is a problem reported. You can do this in psql by using the ["\watch command"](http://paquier.xyz/postgresql-2/postgres-9-3-feature-highlight-watch-in-psql/).

```
\watch [ seconds ]
```

The watch command will re-execute your last SQL every specified number of seconds.
I typically do \watch 5 (to re-execute SHOW POOLS every 5 seconds).
