---
layout: post
title: Elixir/Ecto - update_all with join
date: 2017-10-13 08:53:13
description: How to use update_all with join in Elixir
categories: Elixir
tags: Elixir
---

When I first began using Ecto it was only to directly issue SQL since the PostgreSQL database
that I was working on was so far outside the norm (tables stored in different schemas, different
naming conventions on primary keys, character fields being used to store foreign keys that pointed
at multiple tables). The second project that I worked on was a database that I controlled so
I got to use the more general features of the Ecto query language. The way queries are constructed
is - for the most part - easy to grasp but there are definitely some cases that are harder to understand
and that could benefit from more examples. The Ecto update_all is one of those.

Its a common problem to want to update a set of rows to set a column to a particular value. The
[Ecto update_all](https://hexdocs.pm/ecto/Ecto.Repo.html#c:update_all/3) is the 'Ecto Way' to do
this. Updating a column value for every row in a table is very straightforward.

```
MyRepo.update_all(Post, set: [title: "New title"])
```

So we have a table represented by the Post module that has a column title and this Ecto statement
will result in : UPDATE posts set title = 'New title'. Easy to grasp and it reads really nicely.
Operations to modify each row to a slightly different value are available as well.

```
MyRepo.update_all(Post, inc: [visits: 1])
```

will update Post by incrementing the visits column for each row.
But what if we want to an update_all where we need to join with a number of other tables. This
gets a bit trickier and there are no examples in the doc. We know that the first parameter to
update_all is an Ecto.Queryable so somehow we have to form a query that update_all can deal with.

The current project I'm working on is AWS related and provided an opportunity to do a more
complex update_all. The background on this project is that we wanted to fill in some capabilities
that are lacking in AWS Console so you'll see tablenames that are pretty direct references to
AWS concepts (instances, regions, accounts). Here's the migrate scripts defining the tables of
interest for the update_all example.

```
    create table(:accounts) do
      add :account_name, :string

      timestamps()
    end

    create table(:regions) do
      add :region_name, :string
      add :region_endpoint, :string

      timestamps()
    end

    create table(:instances) do
      add :fqdn, :string
      add :public_ip, :string
      add :private_ip, :string
      add :launch_time, :utc_datetime
      add :instance_id, :string
      add :instance_state, :string
      add :key_name, :string
      add :monitoring, :boolean
      add :instance_type, :string
      add :deleted, :boolean, default: false, null: false
      add :private_cloud, :string
      add :availability_zone, :string

      add :region_id,  references(:regions)
      add :account_id, references(:accounts)

      timestamps()
    end

    create table("jobs") do
      add :job_name, :string, null: false
      add :description, :string
      add :status, :string, null: false
      add :job_type, :string, null: false
      add :alarm_profile_id, references(:alarm_profiles)

      timestamps()
    end

    create table("ec2_job_items") do
      add :instance_id, references(:instances), null: false
      add :job_id, references(:jobs, on_delete: :delete_all), null: false
      add :sns_arn, :string, size: 2048
      add :status, :string, null: false
      add :status_details, :map

      timestamps()
    end
```

So, jobs contain ec2_job_items which are EC2 instances. In AWS an instance
belongs to a region (like us-east-1 or us-west-2) and an account. I needed to
update the ec2_job_items table to set
the sns_arn for every instance that was part of a particular region for a
particular job. The way that I ended up doing this was to form a subquery that
would get me the set of ids for ec2_job_items and then forming the update_all
from there. The subquery was constructed based on a job id and a region_name.
That looked like this:

```
  defp job_item_region_subquery(id, region_name) do
    query = from j in Ec2JobItem,
    join: i in Instance,
    where: j.instance_id == i.id,
    join: r in Region,
    where: i.region_id == r.id,
    where: j.job_id == ^id and r.region_name == ^region_name,
    select: j.id
  end
```

The public function built the remainder of the update_all. It ended up
looking like this:

```
  @doc """
    Set the sns_arn for each job item in a region.
  """
  def update_job_items_srn(id, region_name, sns_arn) do
    query = job_item_region_subquery(id, region_name)
    Repo.update_all(from(j in Ec2JobItem,
                    join: s in subquery(query), on: s.id == j.id),
                    set: [sns_arn: sns_arn])
  end
```

So, you can see that we're joining the table itself to the subquery
based on the ids.

Hopefully this has enough details to help someone else looking for a
good example of a more complex update_all.
