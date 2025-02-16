---
layout: post
title: Elixir/Ecto migrate and rollback
date: 2016-09-03 08:53:13
description: Exploring Features of Ecto - Elixir's Database Library
categories: Elixir
tags:
---

Defining my own database schema for a simple Phoenix/Elixir project to present
a company dashboard. I started with the following objects that I wanted to store
in the relational database :

- Accounts
- Users
- Roles

A user belongs to an account and a user can have one or more roles. I created
the following migration files (there was a single file per table) to accomplish this :

```
    # Accounts table
    defmodule Dashboard.Repo.Migrations.CreateAccount do
      use Ecto.Migration

      def change do
        create table(:accounts) do
          add :name, :string

          timestamps
        end
      end
    end

    # Users table
    defmodule Dashboard.Repo.Migrations.CreateUser do
      use Ecto.Migration

      def change do
        create table(:users) do
          add :name, :string
          add :username, :string, null: false
          add :email_address, :string
          add :password_hash, :string

          add :account_id, references(:accounts, on_delete: :nothing)
          timestamps
        end

        create unique_index(:users, [:username])
      end
    end

    # Roles table
    defmodule Dashboard.Repo.Migrations.CreateRole do
      use Ecto.Migration

      def change do
        create table(:roles) do
          add :name, :string

          timestamps
        end
      end
    end

    # User_roles (the association from a user to possibly more than one role)
    defmodule Dashboard.Repo.Migrations.CreateUserRole do
      use Ecto.Migration

      def change do
        create table(:user_roles) do
          add :user_id, references(:users)
          add :role_id, references(:roles)

          timestamps
        end
      end
    end
```

Since I'm just starting out I sort of stumbled to this set of migrations so I had
to undo this more than once. Ecto and mix make this fairly painless during development.
I can apply the migrations with :

```
    ▶ mix ecto.migrate

    16:21:05.920 [info]  == Running Dashboard.Repo.Migrations.CreateAccount.change/0 forward

    16:21:05.920 [info]  create table accounts

    16:21:05.951 [info]  == Migrated in 0.0s

    16:21:05.998 [info]  == Running Dashboard.Repo.Migrations.CreateUser.change/0 forward

    16:21:05.998 [info]  create table users

    16:21:06.002 [info]  create index users_username_index

    16:21:06.003 [info]  == Migrated in 0.0s

    16:21:06.020 [info]  == Running Dashboard.Repo.Migrations.CreateRole.change/0 forward

    16:21:06.020 [info]  create table roles

    16:21:06.022 [info]  == Migrated in 0.0s

    16:21:06.035 [info]  == Running Dashboard.Repo.Migrations.CreateUserRole.change/0 forward

    16:21:06.035 [info]  create table user_roles

    16:21:06.038 [info]  == Migrated in 0.0s
```

And then checking Postgres using psql I see :

```
    ▶ psql dashboard_dev
    psql (9.5.3)
    Type "help" for help.

    dashboard_dev=# \dt public.*
                   List of relations
     Schema |       Name        | Type  |  Owner
    --------+-------------------+-------+----------
     public | accounts          | table | postgres
     public | roles             | table | postgres
     public | schema_migrations | table | postgres
     public | user_roles        | table | postgres
     public | users             | table | postgres
    (5 rows)
```

The schema_migrations table contains all the migrations that have been applied
by the ecto.migrate task to the database.

```
    dashboard_dev=# select * from schema_migrations;
        version     |     inserted_at
    ----------------+---------------------
     20160904190036 | 2016-09-04 20:21:05
     20160904190150 | 2016-09-04 20:21:06
     20160904190622 | 2016-09-04 20:21:06
     20160904194651 | 2016-09-04 20:21:06
    (4 rows)
```

The version is a generated number based on the current timestamp. To rollback
you can do :

```
    ▶ mix ecto.rollback

    16:24:00.111 [info]  == Running Dashboard.Repo.Migrations.CreateUserRole.change/0 backward

    16:24:00.111 [info]  drop table user_roles

    16:24:00.115 [info]  == Migrated in 0.0s
```

And Ecto will rollback the last migration. To rollback everything applied to the
database you can do :

```
    ▶ mix ecto.rollback --all

    16:24:39.191 [info]  == Running Dashboard.Repo.Migrations.CreateRole.change/0 backward

    16:24:39.191 [info]  drop table roles

    16:24:39.194 [info]  == Migrated in 0.0s

    16:24:39.223 [info]  == Running Dashboard.Repo.Migrations.CreateUser.change/0 backward

    16:24:39.223 [info]  drop index users_username_index

    16:24:39.224 [info]  drop table users

    16:24:39.225 [info]  == Migrated in 0.0s

    16:24:39.241 [info]  == Running Dashboard.Repo.Migrations.CreateAccount.change/0 backward

    16:24:39.241 [info]  drop table accounts

    16:24:39.242 [info]  == Migrated in 0.0s
```

You can now see that all the tables are gone with :

```
    ▶ psql dashboard_dev
    psql (9.5.3)
    Type "help" for help.

    dashboard_dev=# \dt public.*
                   List of relations
     Schema |       Name        | Type  |  Owner
    --------+-------------------+-------+----------
     public | schema_migrations | table | postgres
    (1 row)

    dashboard_dev=# select count(*) from schema_migrations ;
     count
    -------
         0
    (1 row)
```

Other use cases are covered in the documentation for Ecto.Migrate and Ecto.Rollback.
