---
layout: post
title: Config Files in Elixir
date: 2025-03-06 13:30:00
description: Review of How Config Files Are Used in a Phoenix Application
tags:
categories: elixir
giscus_comments: true
---

There are 5 config files generated when you create a new Elixir Phoenix application (using mix phx.new). These are the files and some general information about them.

- config.exs - used for storing global information needed in any environment
- dev.exs - used for storing global information needed only in a dev environment
- prod.exs - used for storing global information needed only in a prod environment
- runtime.exs - used to store global information for any environment.
- test.exs - used for storing global information needed only in a test environment

## When is Evaluation Done

The config.exs, dev.exs, prod.exs and test.exs files are evaluated at build time - before the application is compiled and before dependencies are loaded. Its important to understand this. If you attempt to do something like this in your prod.exs file:

```
config :bureau, Bureau.SecretHolder,
    secret1: System.get_env("SECRET_CONFIG_THING1")
```

Then secret1 will be set to whatever you have set in your compile environment (probably nothing).

If you want to use the system environment to configure how your production app works then you use the runtime.exs file. This file is read after our application and dependencies are compiled and therefore it can configure how our application works at runtime. If the config above was in runtime.exs then secret1 is set to whatever the environment variable SECRET_CONFIG_THING1 stores on the machine on which you run your app.

Be aware that runtime.exs is evaluated at runtime for all the Mix environments. When you are running unit tests locally your runtime.exs file is evaluated.

Because this is the case you'll find code like this in a lot of runtime.exs files:

```
import Config

if config_env() == :prod do
  database_url =
    System.get_env("DATABASE_URL") ||
      raise """
      environment variable DATABASE_URL is missing.
      For example: ecto://USER:PASS@HOST/DATABASE
      """
end
```

We're protecting setting up the database_url to when the config environment is `:prod`. This keeps us from overwriting the quite different database_url we most likely have for dev and test environments.

## What Config Files am I using?

The config.exs file is evaluated (at compile time) for all environments. The dev, test, and prod config files are evaluated based on the environment being compiled. This is because of the line at the end of the config.exs file:

```
import_config "#{config_env()}.exs"
```

The `config_env/0` function is in the Config module in Elixir. It returns the Mix environment. The Mix environment is obtained with:

```
String.to_atom(System.get_env("MIX_ENV") || "dev")
```

That is, Elixir looks for the value associated with "MIX_ENV" and if its not set then its going to default to "dev".

The runtime.exs file, as indicated earlier, is loaded for all environments. Generally, when you are developing locally you are using:

- config.exs
- dev.exs
- runtime.exs

When you are running unit tests you are using:

- config.exs
- test.exs
- runtime.exs

When you are running your code in production you are using:

- config.exs
- prod.exs
- runtime.exs
- (and maybe releases.exs)

## What about releases.exs?

Before Elixir 1.11 an app that wanted to use environment variables in production would use a file called releases.exs. This file was also stored in the config directory. There was no runtime.exs.

The releases.exs was similar to runtime.exs but not quite the same. The releases.exs was evaluated at runtime but only for `:prod`. The runtime.exs is used for all the mix environments.

You can still have a releases.exs (as well as a runtime.exs) in an app. However, its confusing and you should migrate whatever is in your releases.exs file to the runtime.exs file and delete the releases.exs file.

If you do have both files then the runtime.exs file is executed first followed by the releases.exs file. Its easy to see how confusing and error prone it is to have both. Your runtime.exs might try to setup an application environment value and releases.exs could overwrite its value.

## Can a Dependency Function be called from a config file?

Not in any of the config files evaluated at build time. This is because those files are executed prior to loading dependencies. Your dependency code isn't there.

## Can a Dependency Function be called from runtime.exs?

Yes. Keep in mind that the runtime.exs file is not available to whatever code you are calling. In general, it's not a great idea and I'd avoid it.

If you think you absolutely must then I'd keep José Valim's PR notes when introducing runtime.exs in mind:

> Since "config/runtime.exs" is used by both Mix and releases, it cannot
> configure :kernel, :stdlib, :elixir, and :mix themselves. Attempting to
> configure those will emit an error.
> For those rare scenarios, you will need to use
> "config/releases.exs" - but "config/releases.exs"
> will remain simple, which will reduce the odds
> of syntax errors.
> Since "config/runtime.exs" is used by both Mix
> and releases, it cannot invoke "Mix" directly.
> Therefore, for conditional environment compilation,
> we will add a env/2 macro to Config that will be
> available for all config files.
