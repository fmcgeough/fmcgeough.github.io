---
layout: post
title: Elixir, Ecto, Subqueries, GROUP BY
date: 2018-03-04 08:53:13
description: Exploring subqueries and group by with Elixir's ecto library
categories: Elixir
tags:
---

Since I'm very familiar with SQL, if I have to do something a bit
more complicated with Ecto I start with the SQL and then work
towards figuring out how to translate it to Ecto. It's good to
keep in mind that if the going gets too rough with the translation
you are always left with the "out" of just executing raw SQL.
You can map the result set into a schema and from the perspective
of a programmer using the API it looks just the same.

The last query I had that was somewhat more complicated involved a
part of our AWS Detective app that monitors limits on our AWS account.
Users can set thresholds and be sent emails when the limits for
a particular service/resource is a high enough percent of the maximum
allowed. This allows them to open up a case with Amazon to increase
the maximum. We wanted to define one percentage limit as a warning and
another as a critical.

The thing is, we don't want to bombard somebody
with emails. Once we've sent them an email indicating that their threshold
is exceeded (for either warning or critical) we do not want to send them
another one for at least a day - unless the percentage moves between
states. That is, once a service/resource is warning we send an email and
if the service/resource goes into critical we'd send an email. Likewise
we'd send an "all-clear" email if a service was previously in a warning
or critical state and is now below the defined percentages.

As a final requirement, if 24 hours go by and the resource is still
in warning/critical we'd like to send another email to remind the
receiver that there is still an issue and they should take a look at it.

For storage of emails sent we defined a limit_check_threshold_emails
table. The structure of this table is:

```
          Column          |            Type             |
--------------------------+-----------------------------
 id                       | bigint                      |
 email                    | character varying(255)      |
 limit_check_threshold_id | bigint                      |
 previous_state           | character varying(10)       |
 current_state            | character varying(10)       |
 percent_reported         | integer                     |
 inserted_at              | timestamp without time zone |
 updated_at               | timestamp without time zone |
```

So we're storing the email address, naturally, and the association to
the limit_check_threshold the user has defined. We also store the
current_state, the previous state and what percent the resource was
at when we emailed. We have the following states defined: "ok",
"warning", "critical". Storing these as states in db along with the
percent_reported allows us to show somewhat interesting email history.

Since we send emails on transitions (warning to critical) we could have
multiple emails for the same LimitCheckThreshold for a single day. In
order to make decisions on whether to send an email or not we really
need the last email we sent within the previous 24 hours. In SQL we
could get this with:

```
select max(inserted_at) as max_inserted_at,
email,
limit_check_threshold_id
FROM limit_check_threshold_emails
where inserted_at > now() - interval '24 hour'
GROUP BY email, limit_check_threshold_id;
```

and then select rows using these values via a join. So how do we
translate this to Ecto? Its best to take a piece of a time and
build it up. So, lets express the query above in Ecto.

```
def recent_email_group do
    from(
      e in LimitCheckThresholdEmail,
      select: %{
        max_inserted_at: max(e.inserted_at),
        limit_check_threshold_id: e.limit_check_threshold_id,
        email: e.email
      },
      group_by: [e.limit_check_threshold_id, e.email],
      where: e.inserted_at > fragment("now() - interval '24 hours'")
    )
end
```

You can see we used a fragment to express the time frame that we
want to examine. We could have calculated this time in code and used it
directly as a substitution parameter in the code but we chose to use
a fragment. We used a map style select so we have "named" values we
can refer to when we get to having to somehow join this in order to
get our LimitCheckThresholdEmail rows.

Ok, so this gives us the same set as the SQL above. Now, how do we get
the actual LimitCheckThresholdEmail rows in our table? Well, to do that
we'll need to join.

```
  def list_recent_threshold_emails do
    from(
      e in LimitCheckThresholdEmail,
      join: s in subquery(recent_email_group()),
      on:
        s.max_inserted_at == e.inserted_at and s.email == e.email and
          s.limit_check_threshold_id == e.limit_check_threshold_id,
      join: t in assoc(e, :limit_check_threshold),
      preload: [limit_check_threshold: t],
      where: e.inserted_at > fragment("now() - interval '1 day'")
    )
    |> Repo.all()
  end
```

This generates the following SQL:

```
SELECT l0."id", l0."previous_state", l0."current_state",
l0."email", l0."percent_reported", l0."inserted_at",
l0."updated_at", l0."limit_check_threshold_id", l2."id",
l2."warning_percent_threshold", l2."critical_percent_threshold",
l2."emails", l2."service", l2."resource",
l2."inserted_at", l2."updated_at", l2."account_id"
FROM "limit_check_threshold_emails" AS l0
INNER JOIN (SELECT max(l0."inserted_at") AS "max_inserted_at",
 l0."limit_check_threshold_id" AS "limit_check_threshold_id",
 l0."email" AS "email"
 FROM "limit_check_threshold_emails" AS l0
 WHERE (l0."inserted_at" > now() - interval '24 hours')
 GROUP BY l0."limit_check_threshold_id", l0."email") AS s1
 ON ((s1."max_inserted_at" = l0."inserted_at")
 AND (s1."email" = l0."email"))
 AND (s1."limit_check_threshold_id" = l0."limit_check_threshold_id")
 INNER JOIN "limit_check_thresholds" AS l2
 ON l2."id" = l0."limit_check_threshold_id"
 WHERE (l0."inserted_at" > now() - interval '1 day')
```

You might wonder, couldn't we have just selected all the data that
we need in the recent_email_group function and done away with the
need for the subquery? Its a good thought but not in this case. Since
we need the state information and we want the percent_reported
for the emails we'll craft we needed to use the subquery so we can
get the whole row. Using GROUP BY with these additional fields would
pull in more than one row for the service/resource/email combination.
