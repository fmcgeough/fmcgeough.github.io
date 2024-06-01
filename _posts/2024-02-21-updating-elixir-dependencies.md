---
layout: post
title: Elixir and Dependencies
date: 2024-02-21 08:53:13
description: Keeping Your Elixir Dependencies Up To Date
categories: elixir
tags: mix maintenance
giscus_comments: true
---

One of the ongoing tasks for a development team is to keep the dependencies of
apps up to date. The following information are aspects of that work for Elixir
language projects. It doesn't attempt to explain the language itself and assumes
you are at least minimally familiar with it. If you are coming to the language
for the first time then the [Elixir website](https://elixir-lang.org/) has
excellent material to learn the basics of the language.

## Elixir Versioning

Each Elixir library or application has a `mix.exs` file. This file is used to configure
the project. A sample of this is:

```
defmodule Demo.MixProject do
  use Mix.Project

  def project do
    [
      app: :demo,
      version: "0.1.0",
      elixir: "~> 1.11",
      start_permanent: Mix.env() == :prod,
      deps: deps()
    ]
  end

  # Run "mix help compile.app" to learn about applications
  def application do
    [
      extra_applications: [:logger]
    ]
  end

  defp deps do
    [
    ]
  end
end
```

As you can see, a `mix.exs` file is Elixir code. It defines a module and has the
line `use Mix.Project` and some functions. `Mix.Project` expects that there is a
`project/0` function that returns a keyword list representing configuration for
the project.

One of the keys in the sample above is `:version`. This defines the version for
the project. If you generate a new library `mix new my_library` the version is
set to `version: "0.1.0"`. The version key/value must be present. If you try and
remove the version then mix won't be able to compile the code. You'll see this
error: `** (Mix) Please ensure mix.exs file has the :version in the project
definition`.

Elixir requires versions to be in the format `MAJOR.MINOR.PATCH`. Each of these elements
is a number. The meaning of these elements are up to the library developer. However,
most libraries use [Semantic Versioning](https://semver.org/).

## Libraries? Applications?

For Elixir / Erlang libraries and applications can be thought of as the same thing.
It's a named bundle of some functionality with a version. It's definitely helpful to
think about them as the same thing when dealing with versioning and dependencies.

## Semantic Versioning

The [Semantic Versioning website](https://semver.org/) has the definitive explanation
of semantic versioning. You should visit that site for a full explanation.

Important rules for semantic versioning are:

Given a version number MAJOR.MINOR.PATCH, increment the:

- MAJOR version when you make incompatible API changes
- MINOR version when you add functionality in a backward compatible manner
- PATCH version when you make backward compatible bug fixes

One additional rule that is important to keep in mind is: "Major version zero
(0.y.z) is for initial development. Anything MAY change at any time. The public
API SHOULD NOT be considered stable".

## hex.pm

Hex is a package manager for the BEAM ecosystem; any language that compiles to
run on the BEAM VM, such as Elixir and Erlang, can be used to build Hex
packages.

Hex is an open-source [project](https://github.com/hexpm/hex) initiated in
early 2014, and continues to evolve under the stewardship of Six Colors AB which
was founded in 2018 by Hex's creator, Eric Meadows-Jönsson. The project provides
tasks that integrate with the Elixir mix tool.

Hex provides a [website](https://hex.pm/) that allows retrieval of libraries
and their associated documentation by version number. It allows organizations to
setup private package publishing. This means that libraries developed by your
organization could be published to hex.pm and only be available for developers
that are in your organization.

Since Hex is open-source Cloudsmith created their own private hex repository in
February 2014. See the [Cloudsmith blog
post](https://cloudsmith.com/blog/worlds-first-private-hex-repository-with-cloudsmith)
for more information on this.

## Specifying Library Dependencies in mix.exs

You can examine hex.pm to find the current version of a library. One of the keys returned
by the `project/0` function your mix.exs file is `:deps`. This returns a list of dependencies
and is generally seen like above: `deps: deps()`. That is, the function `deps/0` returns
the list of dependencies. Setting up the dependency is adding a line to the list in that
function. For example:

```
  defp deps do
    [
      {:telemetry, "~> 1.2"}
    ]
```

## Organization Libraries Stored "Elsewhere"

Some organizations use github or gitlab to store their organization's Elixir
libraries. Generally those organizations would tag the library code and build releases
for their developers.
