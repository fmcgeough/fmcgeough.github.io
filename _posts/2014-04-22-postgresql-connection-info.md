---
layout: post
title: Postgresql Connection Info
date: 2014-04-22 15:36:50
description: Who is connected to my Postgres database?
tags: postgresql
categories: database
---

Some days you just want to know who is connecting up to your database. The following SQL works for 9.3 Postgresql.

    SELECT COUNT(1) as num_connections,
    COUNT(CASE WHEN s."state" = 'idle' THEN 1 ELSE NULL END) as num_idle_connections,
    COUNT(CASE WHEN s."state" = 'idle in transaction' THEN 1 ELSE NULL END) as num_idle_in_tx_connections,
    max(CASE WHEN s."state" = 'idle in transaction' THEN GREATEST(now(),s.query_start)-s.query_start ELSE NULL END) as age_of_oldest_tx,
    s.datname,
    s.client_addr
    FROM pg_stat_activity s
    WHERE s.pid != pg_backend_pid()
    and s.client_addr is not null
    GROUP BY s.client_addr, s.datname
    ORDER BY 1 DESC;

This query shows you total connections from by ip address along with some info that I find helpful - namely how many connections are connected but idle and how many are idle in transactions (and if so for how long).
