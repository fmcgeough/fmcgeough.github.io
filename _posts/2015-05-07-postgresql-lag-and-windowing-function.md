---
layout: post
title: Postgresql - Lag and Windowing Function
description: Using Lag and window operations with Postgresql
date: 2015-05-07 22:03:34
categories: SQL
tags:
---

Lag came in handy again.

Problem: customer stored their account info in an account_data table and recorded history of the changes to this table in an account_history table along with a new column dt_account_history_updated. Groovy. The way the account_history table was written was :

- on insert - write the values to account_history
- on update - write only the old values to account_history, but only for the column values that changes (using NULLIF function).
- on delete - write values to account_history.
  hmmm. so what you are looking at in account_history is the previous values. not the current ones (for update anyway). Got it?

So... each customer has a status associated with it (ACTIVE, CANCELLED, SUSPENDED, etc). The question is how could you report when a customer was in that status if the account_history has only the previous value? Well. actually pretty easy using LAG.

    with set_of_data as
    (
    SELECT ah.account_id, ah.dt_last_history_updated as dt_info, ah.status
    FROM account_history ah
    WHERE ah.status IS NOT NULL
    UNION
    SELECT acct.account_id, now(), acct.status
    FROM account_data acct
    ),
    build_data_set as
    (
    SELECT account_id, dt_info, status, LAG(dt_info, 1) OVER (PARTITION BY account_id ORDER BY dt_info ASC) as lag
    FROM set_of_data
    )
    SELECT account_id,
    status,
    timezone('GMT'::text, lag) as dt_status
    FROM build_data_set
    WHERE lag IS NOT NULL
    ORDER BY 1, 3;

tada! We get all the data from the history table where status has changed (since we only write on updates when column value changes) and we get current state from live table. Then we select out of this set by use a LAG 1 and order by the dt_info column for each account_id. This ends up putting the previous dt_info column in the row after it historically - which is what tells us exactly when we went to that state.

Reference : [Postgresql 9.3 Window Functions](http://www.postgresql.org/docs/9.3/static/functions-window.html)
