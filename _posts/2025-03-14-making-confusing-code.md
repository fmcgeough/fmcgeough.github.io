---
layout: post
title: Writing Confusing Code in Elixir
date: 2025-03-14 14:11:00
description: Comparison of different ways of writing a controller function
tags:
categories: elixir
giscus_comments: true
---

I got into the Elixir language fairly early on. I was working at a company that used Java and Scala and was pretty bored. I was, in truth, also aggravated at the quality of the code that I'd see produced. The code would solve the problem, true. But it solved it in a way that made the code hard to look at. And hard to look at code is generally not going to be maintained well.

Looking at Elixir code and beginning to write it was great. I thought, foolishly, that here was a language that would eliminate this type of code. It was so easy to write code that was great to read. The fact that it was built around immutability eliminated a whole swath of problems that I'd seen. It was so much easier to write code that solved problems with a collection of worker processes. And, importantly for me, it was easy to read.

I developed a few Phoenix web apps for the company. These were definitely skunk works projects that I developed completely independently. I let engineering teams know that they were available and they got used a fair amount. They became one part of forming a SRE team at the company. After this I moved on to work at companies where Elixir was a first class citizen. That's when I discovered that, as it turns out, just like with Java, you can make interesting messes with Elixir too. Here's one example.

I worked on a service that had a REST API that fetched data. The data could be paged. The caller passed a limit and offset as query parameters. There were a few simple rules:

- limit and offset both had to be integer values (well String representations of those values since they are coming in as query parameters)
- limit had to be greater than or equal to 1
- limit had to be less than or equal to 25
- offset had to be greater than or equal to 0 (no negative offsets)

The code to handle this looked something like this (simplified):

```
def fetch_page(conn, params) do
  Parameters.parse(params, fn limit, offset ->
    case do_some_fetch(limit, offset) do
      {:ok, data} -> send_response(conn, data, limit, offset)
      :error -> invalid_request(conn)
  end)
  rescue e ->
     ## Some more code to handle exceptions that might get thrown
     ## by the Parameters module or data fetching
end
```

That is, the folks that wrote the code created a module to parse out the limit and offset but the function that was written required that the caller pass in a function that the `Parameters` module would call passing it the parsed limit and offset. The actual work is inside this function.

The problem with this code was 1) the wrong module is in the driver's seat. Parameters shouldn't be what is driving how my code works. It makes error handling awkward and the code harder to read; 2) in order to test the Parameters module you have to pass a function to the parse/1 function. That's going to cause initial confusion for developers added to the project. Callbacks like this are not a great idea; 3) once the parameters are parsed the code was then passing the data on to the fetcher as individual parameters. There's nothing intrinsically wrong with this. But it's kind of a pain if we decide to add additional parameters later. I prefer to pass a map that has been typed and documented; 4) Instead of using a FallbackController to handle errors the controller was handling errors. It also was forced to account for exceptions that might be thrown by the Parameters parse or the data fetching. I don't want this in my controller code. I want the lower level code to handle these problems so that my controller code is clean and straightforward. Instead of that we had code that was kind of dense and hard to parse at a glance.

So I refactored this into something that looked more like this:

```
def fetch_page(conn, params) do
    with {:ok, parsed_params} <- Parameters.parse(params),
         {:ok, fetched_data} <- DataFetcher.fetch(parsed_params) do
      send_response(conn, parsed_params, fetched_data)
    end
end
```

Using the with statement means we only send a response if the caller of the REST API passed valid parameters and the code was able to get to the database and fetch data.

In order to get to this state there had to be some thought given to the errors returned by the modules called by the controller. We had to normalize on an error format and agree on a set of standard errors. This was easy to do and made everyone happier. The code was easy to look at, test and maintain.

One thing to keep in mind when using the `with` approach is that the errors should be distinctive to allow separation of "the caller passed us invalid parameters" vs "the caller parameters were fine but we had an issue fetching data". This is important because you want the user of the REST API to know whether they are doing the right thing (even if the service wasn't able to do what they wanted).

When you write code try and imagine what it is going to be like to maintain the code (whether this is you or someone a few years later who has no idea what you had in mind).
