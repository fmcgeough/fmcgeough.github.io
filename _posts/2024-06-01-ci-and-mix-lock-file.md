---
layout: post
title: Continuous Integer, Elixir and mix.lock file
date: 2024-06-01 11:30:00
description: Ensure your mix.lock file is not changing without your knowledge
categories: elixir
tags: mix
giscus_comments: true
---

Elixir's mix.lock file is used to store information on all the dependencies
for your application. It should be checked into source control just like
your mix.exs file.

If you are coming from other languages you probably have seen this same concept.
For example, in Python there is
the [Pipfile and the Pipfile.lock](https://github.com/kennethreitz/pipenv). In Ruby
there is the [Gemfile and the Gemfile.lock](https://bundler.io/). In Javascript
there is the [package-lock.json](https://docs.npmjs.com/).

I recommend for continuous integeration for Elixir you should use the command:

```
mix deps.get --check-locked
```

This is available in v1.15.0 and greater. It will raise if there are pending
changes to the lockfile. This works to ensure the lock file that you check into
source control is the one that is being used to build your released software.

See the documentation at https://hexdocs.pm/mix/Mix.Tasks.Deps.Get.html.
