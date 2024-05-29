---
layout: post
title: Elixir, Ecto, embeds_many
date: 2017-04-29 08:53:13
description: Features of Ecto - Elixir's database library
categories: Elixir
tags: Elixir
---

I built an internal project using Elixir and the Phoenix web framework. We use
AWS and I wanted to build our own site that could quickly display EC2 instance
information along with additional information that we'd gather from our deployment
process. We wanted to add some initial simple functionality as well like searching
within the attributes on an instance instead of just across all attributes that
AWS Console provides and some various quick "copy to clipboard" type functionality
that were deemed important for operations. I named the app - aws_detective.

I decided to store the data in a Postgres instance and use [Ecto](https://github.com/elixir-ecto/ecto)
for handling interaction with Postgres.

Because the app's purpose was to provide EC2 instance information I created an
"instances" table with columns defined for the major information that we wanted to
display like instance_id, public_ip, private_ip, etc. Nothing very remarkable about
the data types themselves and using Ecto for this type of thing is well covered in
the Ecto documentation and various blog posts.

When the Phoenix app started I'd start a supervised Elixir process that would run in the
background and periodically obtain new EC2 instance information and perform a diff
between what was obtained and what was currently stored in Postgres and do the correct
set of inserts, updates and deletes (although in our case deletes were actually just
marking the instance row as deleted so that we could analyze instance destruction trends).

EC2 Instance information contains tags. Tags are simply a key and a value associated
with an instance. You can have many of these set on an instance and this metadata can
be used for various EC2 tasks. I decided to store the
tags as an embedded object in my "instances". Since there are numerous tags on an instance
I decided to use the [embeds_many](https://hexdocs.pm/ecto/Ecto.Schema.html#embeds_many/3) approach. This did not seem to be as well documented (although part of that is me not knowing exactly where to look)
so I figured I'd write a short blog post in case someone else decides to do the same thing
and is a bit confused. Hopefully if you arrive to read this for that purpose I don't further muddy the waters.

An embeds_many is defined in Postgres as a jsonb[]. That is, an array of JSONB. Using embeds_many
for this particular purpose means that I could map the JSON array into an Array of Elixir
schema structures. Here's how I proceeded :

The migration file. To setup a jsonb[] column for the table I provided the following definition :

```
    def change do
      alter table(:instances) do
        add :tags, {:array, :map}, default: []
      end
    end
```

The mapping to Elixir was defined in a tag.ex file as :

```
    defmodule AwsDetective.Tag do
      use AwsDetective.Web, :model

      @primary_key false
      embedded_schema do
        field :key, :string
        field :value, :string
      end

      def changeset(struct, params \\ %{}) do
        struct
        |> cast(params, [:key, :value])
        |> validate_required([:key])
      end
    end
```

This indicates that we don't want a primary key (generated sequence number) nor timestamps
fields in our JSON. Our instances schema (just the definition of the tags column was defined as :

```
    schema "instances" do

      ...

      embeds_many :tags, Tag, on_replace: :delete
    end
```

The on_replace macro was important. If any of the tags change we are just going to replace all of them
with whatever the new tags are. We are not going to attempt to "update" individual elements within the
tags column. Here's the relevant section from the Ecto doc :

> The embedded may or may not have a primary key. Ecto use the primary keys to detect if an embed is being updated or not. If a primary is not present and you still want the list of embeds to be updated, :on_replace must be set to :delete, forcing all current embeds to be deleted and replaced by new ones whenever a new list of embeds is set.

The Ecto.Changeset has further enlightening information on the options for embeds.

- :raise (default) - do not allow removing association or embedded data via parent changesets
- :mark_as_invalid - if attempting to remove the association or embedded data via parent changeset - an error will be added to the parent changeset, and it will be marked as invalid
- :nilify - sets owner reference column to nil (available only for associations)
- :update - updates the association, available only for has_one and belongs_to. This option will update all the fields given to the changeset including the id for the association
- :delete - removes the association or related data from the database. This option has to be used carefully

I believe :nilify and :delete amount to the same thing in my case but I'm not sure. I'll have to try each option
out and try and puzzle out what they do.

The changeset method on AwsDetective.Instance became :

```
    def changeset(struct, params \\ %{}) do
      struct
      |> cast(params, [... long list of EC2 instance columns...])
      |> cast_embed(:tags)
    end
```

This is how you get an embedded schema into a changeset. The params map has a key :tags where
the value is an array of structures each representing a Tag, that is, a :key and a :value. The
cast_embed in Instance will in turn call the Tag changeset. At least that is what I understand
at this point. In any case, this allowed the :embeds_many to function.

There are downsides to using :embeds_many. You probably want to store the data as just jsonb
and not jsonb[] if you are wanting to do searches. The addition of jsonb to Postgres provides
some interesting index options that can provide really quick search results with this. I'm less
clear on how to do this with an array of jsonb. Its most likely possible but I haven't looked
into this and it wasn't necessary for this project. jsonb itself could contain an array (obviously)
and the data could be stored in json and manipulated that way in the code.

`:embeds_many` seems like a good candidate if there is N rows of additional data that you want to
store on a row in a table and this additional data doesn't seem to warrant creating and managing a separate table.
In addition, its probably most useful if the structure is unlikely to change. I thought this was the case for this particular problem. Ecto provides the mapping to your defined structure and then its easy to manipulate
to render JSON or HTML for a client.

The only other thing I'd like to point out is that I used
the excellent [ex_aws](https://github.com/CargoSense/ex_aws)
library. This is by far the most downloaded library for dealing with Amazon API's in Elixir.
If you have a need to do this I'd recommend downloading and trying that library.
