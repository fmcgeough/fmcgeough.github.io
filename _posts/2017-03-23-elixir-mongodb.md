---
layout: post
title: Elixir, Mongo, Regex
date: 2017-03-23 08:53:13
description: Accessing MongoDB with Elixir
categories: Elixir
tags:
---

I built a Phoenix app for my company that provides lots of administrative insight into
our system. It gives operations and support folks some information beyond what's provided
in our traditional UI. It also tries to diagnose common problems and provide suggestions on
how they can be fixed (or indications of what else to examine).

The majority of the data in our system is stored in Postgres. That worked really well for
me in using Phoenix and Elixir since Postgres is very well supported
by the [Postgrex driver](https://hex.pm/packages/postgrex).
I haven't had any issues with it. So, many thanks to everyone that worked on that.

I had a new request to provide a page that would have to read from a Mongo database. I
hadn't explored the Mongo driver situation in Elixir before. I got it to work fairly
quickly but its a lot less smooth and seemingly less mature than Postgrex. That may just
be my lack of exposure to Mongo in general. I've always lived in the relational database
world and although I've played with Mongo I've never looked at it for anything serious
and didn't know it nearly as well as I probably should have. This post is about various
things I found out. Some of them (or maybe all of them) may be no big news for folks that
use Mongo on a regular basis. But if you are more like me and don't know that much it might
be helpful. As in the case of the Postgres driver many thanks to all the developers that
have contributed their time and effort to get the Mongo driver to its current state.

[hex.pm](https://hex.pm/) is the repo for Elixir. Its fairly awesome. One thing that's
great about this is you can easily see what is the most popular library for what you are
interested in. In the case of Mongo it looks like the clear favorite is [mongodb](https://hex.pm/packages/mongodb).
The latest version is 0.2.0 created November 11, 2016.

I created a sample project to fiddle around with :

```
mix new mongo_test
```

Then I edited the mix.exs file to add the dependencies suggested in the Mongodb doc :

```
defp deps do
  [{:mongodb, "~> 0.2.0"},
   {:poolboy, ">= 0.0.0"}]
end
```

I had a mongo instance running locally. I populated a new collection with some test data :

```
db.testData.insert({"x" : "Hello" })
db.testData.insert({"x" : "Goodbye" })
db.testData.insert({"x" : "Frank" })
db.testData.insert({"x" : "Fred" })
db.testData.insert({"x" : "Four" })
db.testData.insert({"x" : "ourF" })
db.testData.insert({"x" : "ourf" })
db.testData.insert({"x" : "Foot" })
db.testData.insert({"x" : "123324"})
db.testData.insert({"x" : "123324 ABC"})
db.testData.insert({"x" : "frank"})
```

and then I fired up iex with iex -S mix to try and connect to the database and
search this collection. Starting a process that provides the connection to Mongo
is straightforward. You can name the pid with an atom so that you can reference
that in the future instead of keeping the pid around and having to manage it.

```
iex(1)> {:ok, _} = Mongo.start_link(database: "test", name: :mongo)
iex(2)> :mongo |> Mongo.find("testData", %{}) |> Enum.to_list()
  [%{"_id" => #BSON.ObjectId<58d4b1d6a6ba445542adc9d3>, "x" => "Hello"},
   %{"_id" => #BSON.ObjectId<58d4b1d6a6ba445542adc9d4>, "x" => "Goodbye"},
   %{"_id" => #BSON.ObjectId<58d4b1d6a6ba445542adc9d5>, "x" => "Frank"},
   %{"_id" => #BSON.ObjectId<58d4b1d6a6ba445542adc9d6>, "x" => "Fred"},
   %{"_id" => #BSON.ObjectId<58d4b1d6a6ba445542adc9d7>, "x" => "Four"},
   %{"_id" => #BSON.ObjectId<58d4b1d6a6ba445542adc9d8>, "x" => "ourF"},
   %{"_id" => #BSON.ObjectId<58d4b1d6a6ba445542adc9d9>, "x" => "ourf"},
   %{"_id" => #BSON.ObjectId<58d4b1d6a6ba445542adc9da>, "x" => "Foot"},
   %{"_id" => #BSON.ObjectId<58d4b1d6a6ba445542adc9db>, "x" => "123324"},
   %{"_id" => #BSON.ObjectId<58d4b1d6a6ba445542adc9dc>, "x" => "123324 ABC"},
   %{"_id" => #BSON.ObjectId<58d4b1d6a6ba445542adc9dd>, "x" => "frank"}]
```

So that seemed pretty straightforward. I then tried to make regular expressions
work. This had me stumped for hours. There is a BSON.Regex in mongodb and I "knew"
I had to use that but if you look at the test cases for mongodb you'll see that
the two tests related to BSON.Regex just form two different ones and then inspect
them to ensure they are setup properly.

```
test "inspect BSON.Regex" do
  value = %BSON.Regex{pattern: "abc"}
  assert inspect(value) == "#BSON.Regex<\"abc\">"

  value = %BSON.Regex{pattern: "abc", options: "i"}
  assert inspect(value) == "#BSON.Regex<\"abc\", \"i\">"
end
```

so I tried to do something like this :

```
iex(2)> :mongo |> Mongo.find("testData", %{x: %BSON.Regex{pattern: "F"}}) |> Enum.to_list()
iex(3)> :mongo |> Mongo.find("testData", %{x: %BSON.Regex{pattern: "F"}}) |> Enum.to_list()
** (ArgumentError) argument error
                    :erlang.iolist_size(["", 11, ["x", 0], [["F", 0], nil, 0]])
          (mongodb) lib/bson/encoder.ex:108: BSON.Encoder.document/1
           (elixir) lib/enum.ex:1229: Enum."-map/2-lists^map/1-0-"/2
    (db_connection) lib/db_connection.ex:1001: DBConnection.encode/4
    (db_connection) lib/db_connection.ex:635: DBConnection.execute/4
          (mongodb) lib/mongo.ex:368: Mongo.raw_find/5
          (mongodb) lib/mongo/cursor.ex:34: anonymous fn/6 in Enumerable.Mongo.Cursor.start_fun/6
           (elixir) lib/stream.ex:1242: anonymous fn/5 in Stream.resource/3
```

Well, nope. I'm doing something wrong but what? The problem is that the definition of
BSON.Regex doesn't define default values for the two parts of the defstruct : pattern and
options. But, the code that uses BSON.Regex expects that both of them will be there. So,
the following works fine :

```
iex(4)> :mongo |> Mongo.find("testData", %{x: %BSON.Regex{pattern: "F", options: ""}}) |> Enum.to_list()
[%{"_id" => #BSON.ObjectId<58d4b1d6a6ba445542adc9d5>, "x" => "Frank"},
 %{"_id" => #BSON.ObjectId<58d4b1d6a6ba445542adc9d6>, "x" => "Fred"},
 %{"_id" => #BSON.ObjectId<58d4b1d6a6ba445542adc9d7>, "x" => "Four"},
 %{"_id" => #BSON.ObjectId<58d4b1d6a6ba445542adc9d8>, "x" => "ourF"},
 %{"_id" => #BSON.ObjectId<58d4b1d6a6ba445542adc9da>, "x" => "Foot"}]
```

You can see from the results that what this did was to match on any document in our collection that
had a "F" character in the "x" field (regardless of its position).

What are the options? What do they do? The best doc I could find was on the godoc.org site.

"RegEx represents a regular expression. The Options field may contain individual characters defining the way in which the pattern should be applied, and must be sorted. Valid options as of this writing are 'i' for case insensitive matching, 'm' for multi-line matching, 'x' for verbose mode, 'l' to make \w, \W, and similar be locale-dependent, 's' for dot-all mode (a '.' matches everything), and 'u' to make \w, \W, and similar match unicode. The value of the Options parameter is not verified before being marshaled into the BSON format."

I'm not sure that these flags are applicable. It looks like the pattern in this case is always a
regex that is compiled. I can do :

```
iex> :mongo |> Mongo.find("testData", %{x: %BSON.Regex{pattern: "Frank|Fred", options: ""}}) |> Enum.to_list()
[%{"_id" => #BSON.ObjectId<58d49868a6ba445542adc9c9>, "x" => "Frank"},
 %{"_id" => #BSON.ObjectId<58d49868a6ba445542adc9ca>, "x" => "Fred"}]
```

and we find both the row with Frank and the row with Fred. The "|" character is a logical OR. I can change this
to match case-insensitive by passing "i" as an option. This does seem to have a useful impact.

```
iex> :mongo |> Mongo.find("testData", %{x: %BSON.Regex{pattern: "Frank|Fred", options: "i"}}) |> Enum.to_list()
[%{"_id" => #BSON.ObjectId<58d49868a6ba445542adc9c9>, "x" => "Frank"},
 %{"_id" => #BSON.ObjectId<58d49868a6ba445542adc9ca>, "x" => "Fred"},
 %{"_id" => #BSON.ObjectId<58d4aa7ea6ba445542adc9d2>, "x" => "frank"}]
```

Passing any string will match a document if the search element occurs anywhere in the string. So,

```
iex > :mongo |> Mongo.find("testData", %{x: %BSON.Regex{pattern: "an", options: ""}}) |> Enum.to_list()
[%{"_id" => #BSON.ObjectId<58d49868a6ba445542adc9c9>, "x" => "Frank"},
 %{"_id" => #BSON.ObjectId<58d4aa7ea6ba445542adc9d2>, "x" => "frank"}]
```

If you just wanted to find the strings that start with "an" then you can use an anchoring character "^"
as in :

```
iex > :mongo |> Mongo.find("testData", %{x: %BSON.Regex{pattern: "^an", options: ""}}) |> Enum.to_list()
[]
```

You can find documents that can contain numbers in the "x" field with :

```
iex >  :mongo |> Mongo.find("testData", %{x: %BSON.Regex{pattern: "[0-9]", options: ""}}) |> Enum.to_list()
[%{"_id" => #BSON.ObjectId<58d4b1d6a6ba445542adc9db>, "x" => "123324"},
 %{"_id" => #BSON.ObjectId<58d4b1d6a6ba445542adc9dc>, "x" => "123324 ABC"}]
```

Since Mongo is using a Perl compatible regex a good resource for simple regex is the
[perldoc](http://perldoc.perl.org/perlrequick.html#Simple-word-matching).

This is about all I've learned about the Elixir Mongo driver so far. I hope to spend some more
time with it over the next few days to learn more about both it and Mongo.
