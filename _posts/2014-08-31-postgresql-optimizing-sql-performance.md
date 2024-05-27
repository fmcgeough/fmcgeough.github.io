---
layout: post
title: Postgresql - Optimizing SQL Performance
date: 2014-08-31 18:08:30
description: SQL Optimization with Postgresql
categories: SQL postgresql
tags: SQL postgresql
---

In a standard smallish Postgresql installation its actually fairly straightforward to figure out what indexes to create to eliminate sequential scans and improve your performance. But in a larger system where there are thousands of different queries, perhaps written by dozens of different engineers, the problem of addressing performance issues gets a bit more difficult. However, the same techniques can be used on both systems. I'll describe what I use and perhaps it will be useful for someone else.

## pgbadger

pgbadger is a [Postgresql log analyzer](https://github.com/dalibo/pgbadger). You should have it installed and setup a cron job to analyze your log every hour and send you a report. This is sort of a default thing to do on any Postgresql installation. It gives some nice general information.

In order to make use of it you'll need to setup your Postgresql installation to actually log. There are settings in postgresql.conf that will need to be configured in the "ERROR REPORTING AND LOGGING" section. Here is what I use :

    log_destination = 'stderr'
    logging_collector = on
    log_filename = 'postgresql-%Y%m%d-%H.log'
    log_truncate_on_rotation = on
    log_rotation_age = 60min
    log_rotation_size = 0
    log_directory = '/postgres/tracelogs/pgsql'
    log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d %h '

I use pgbadger from a shell script (badger_reporting.sh) that has a single line in it :

    # have pgbadger parse our Postgresql log
    /usr/local/bin/pgbadger --prefix '%t [%p]: [%l-1] user=%u,db=%d %h ' --outfile $@
    #

Note: you'll want the line you pass into pgbadger to mirror the output defined in your `log_line_prefix` definition. You are giving pgbadger the information that it needs to properly parse your log file.

This shell script is called from another that passes in the name of the file to generate and the log file as source like this :

    badger_reporting.sh $LOCAL_HTML_OUTFILE $LOCAL_LOG_FILE

Since we're going to send an email to ourselves with our cron job its also nice to grep the postgresql log to see if there are ERRORS in it and send those in the body of the email so its immediately obvious. I do this with :

    echo "Errors in log" > $LOCAL_LOG_FILE".err"
    grep "ERROR:" $LOCAL_LOG_FILE >> $LOCAL_LOG_FILE".err"

The mailing on our system is handled by mutt.

    # mail out the pgbadger output
    cat $LOCAL_LOG_FILE".err" | mutt -s "$SUBJECT" -a "$LOCAL_HTML_OUTFILE" -- "$RECIP_LIST"
    #

There are a lot of sections in the pgbadger report. The top of the report has "Overall Statistics". You might want to familiarize yourself with the numbers that show up here and look for things that are out of the ordinary for the same day and time in your system. I also tend to look at "Queries that took up the most time" section.

Anyway, pgbadger is a useful tool. Its free and fairly easy to setup. On most systems it provides at least some insight.

You should realize that on most systems you'll also have to set the `log_min_duration_statement` to some value that keeps your log from exploding in size. This means that pgbadger will have a whole slew of statements (ordinarily the bulk of them) that it won't know anything about. Keep this in mind. There are other tools you can add to your arsenal that provide more full insight including the one I'll describe next.

## pg stat statements

This should be required for any production Postgresql system (it actually is turned
on if you use Amazon RDS version of Postgresql). If you come to a new system that doesn't have it loaded this should be one of the first things that you address. While you're at it add auto_explain to the set of things that get pre-loaded by postgresql. This is done by editing your postgresq.conf file and doing this :

    # modify what preload libraries Postgresql uses
    shared_preload_libraries = 'auto_explain,pg_stat_statements' # (change requires restart)

As noted in the awesome postgresql.conf doc that accompanies the file, a change to this line requires a Postgresql restart. Do it. Then run the following from psql :

    create extension pg_stat_statements;

You are now much more awesome. As noted on the Postgresql online doc : The `pg_stat_statements` module provides a means for tracking execution statistics of all SQL statements executed by a server. In a sense this module is like a super-sized pgbadger. It gives you information that is immensely valuable. Read all about it on web at : [pg_stat_statements](http://www.postgresql.org/docs/9.3/static/pgstatstatements.html) (change URL for whatever version you happen to be using).

The default number of statements that are tracked is 1,000. Personally I'd bump that up to 10x by setting the following parameter in postgresql.conf file.

    pg_stat_statements.max = 10000

But this is really dependent upon your system and how many different types of queries you have hitting your database.

The important queries to know about `pg_stat_statements` are :

    -- reset statements. Empty everything out and start regathering
    SELECT pg_stat_statements_reset();

    -- which queries were called the most
    SELECT * FROM pg_stat_statements ORDER BY calls desc LIMIT 50;

    -- which queries used the most CPU time
    SELECT * FROM pg_stat_statements ORDER BY total_time desc LIMIT 50;

    -- what are the top 50 queries that I should look at (by total time hit)?
    SELECT query,
    calls,
    total_time,
    rows,
    100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent,
    ROUND((total_time::numeric / calls::numeric),2) as average_time_per_call
    FROM pg_stat_statements
    ORDER BY total_time
    DESC LIMIT 50;

    -- what are the top 50 queries that I should look at (by speed of execution) ?
    SELECT query,
    calls,
    total_time,
    rows,
    100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent,
    ROUND((total_time::numeric / calls::numeric),2) as average_time_per_call
    FROM pg_stat_statements
    ORDER BY 6
    DESC LIMIT 50;

How often should you run : `pg_stat_statements_reset`? Its really totally dependent upon your system. Here's what I do. Every hour I grab the contents of pg_stat_statements and mail it
to myself. Then I run pg_stat_statements_reset. This gives me a history and an hourly
view of what is going on in the database.

## pg stat user tables

The `pg_stat_user_tables` contains a wealth of statistical information at a low level about your app tables. It is where to go to see if you are consistently sequentially scanning your million row table (whoops!) just because you missed adding an index after an application query change. Since its a running total I recommend taking a snapshot of the data, storing it in a temporary table and then taking another snapshot and diff'ing the two in order to see what happened over the period of time between your two snapshots. I actually have our system setup so this happens every hour during weekdays and business hours. I keep about a month worth of these snapshots in the temporary table. This lets me easily go back to a previous week (or two) and see if today's traffic is significantly different from its corresponding traffic on same day at a point in the past.

To create the temporary table perform the following SQL :

    create table public.history_of_pg_stat as
    SELECT CURRENT_TIMESTAMP as time_captured, *
    FROM pg_stat_user_tables;

Then periodically do the following :

    insert into public.history_of_pg_stat
    SELECT CURRENT_TIMESTAMP, *
    FROM pg_stat_user_tables;

Now you can diff the two captured rows by :

    with most_recent_time as
    (
    SELECT time_captured
    FROM public.history_of_pg_stat
    ORDER BY 1 DESC LIMIT 1
    )
    ,
    second_time as
    (
    SELECT time_captured
    FROM public.history_of_pg_stat
    WHERE time_captured < (SELECT time_captured FROM most_recent_time)
    ORDER BY 1 DESC
    LIMIT 1
    )
    ,
    hops2 as
    (
    SELECT * FROM public.history_of_pg_stat
    WHERE time_captured = (SELECT time_captured FROM most_recent_time)
    ),
    stats as
    (
    SELECT to_char(hops.time_captured at TIME ZONE 'EST5DT', 'HH24:MI:SS') || ' - ' || to_char(hops2.time_captured at TIME ZONE 'EST5DT' , 'HH24:MI:SS') as capture_times,
    hops2.time_captured-hops.time_captured as time_range,
    hops2.schemaname || '.' || hops2.relname as tablename,
    hops2.seq_scan - hops.seq_scan as seq_scans,
    hops2.seq_tup_read - hops.seq_tup_read as seq_tup_read,
    hops2.idx_scan - hops.idx_scan as idx_scans,
    hops2.idx_tup_fetch - hops.idx_tup_fetch as idx_tup_fetch,
    hops2.n_tup_ins - hops.n_tup_ins as n_tup_ins,
    hops2.n_tup_upd - hops.n_tup_upd as n_tup_upd,
    hops2.n_live_tup - hops.n_live_tup as n_new_live_tups,
    hops2.n_live_tup as current_live_tup
    FROM public.history_of_pg_stat hops
    JOIN hops2 ON (hops.schemaname = hops2.schemaname and hops.relname = hops2.relname)
    WHERE hops.time_captured = (SELECT time_captured FROM second_time)
    )
    SELECT * FROM stats
    WHERE (seq_scans > 0 OR idx_scans > 0)
    AND tablename NOT LIKE 'public.history_of_pg_stat'
    AND current_live_tup > 5000
    ORDER BY 5 DESC;

I hope some of this sketched out advice helps someone else in tracking down performance problems or better administer their system. Happy hunting!
