---
layout: post
title: Staying up to date in Elixir
date: 2025-03-02 16:04:00
description: Latest Phoenix and mix phx.new
tags:
categories: elixir
giscus_comments: true
---

One thing I suggest to folks new to Elixir is to periodically get the latest version of `phx.new` and use it to generate a project. Examine the output to see what has changed.

The reason this is somewhat important is to keep up with changes to the ecosystem. The majority of developers are wrapped up in day to day activities. They don't have time to follow issues in github or discussions on mailing list or Elixir's Discord channel.

The steps below were done with Elixir 1.17.2 (compiled with Erlang/OTP 27).

## Current phx.new

You can install phx.new using mix:

```
$ mix archive.install hex phx_new
Resolving Hex dependencies...
Resolution completed in 0.007s
New:
  phx_new 1.7.20
* Getting phx_new (Hex package)
All dependencies are up to date
Compiling 11 files (.ex)
Generated phx_new app
Generated archive "phx_new-1.7.20.ez" with MIX_ENV=prod
```

If there is a current version installed you'll be asked if you wish to overwrite it.

## Generating an API only service

If you are writing a service to provide a JSON API then you can leave out support for HTML, JS, mailer and esbuild.

```
 mix phx.new --no-html --no-live --no-tailwind --no-esbuild --no-mailer bureau
* creating bureau/lib/bureau/application.ex
* creating bureau/lib/bureau.ex
* creating bureau/lib/bureau_web/controllers/error_json.ex
* creating bureau/lib/bureau_web/endpoint.ex
* creating bureau/lib/bureau_web/router.ex
* creating bureau/lib/bureau_web/telemetry.ex
* creating bureau/lib/bureau_web.ex
* creating bureau/mix.exs
* creating bureau/README.md
* creating bureau/.formatter.exs
* creating bureau/.gitignore
* creating bureau/test/support/conn_case.ex
* creating bureau/test/test_helper.exs
* creating bureau/test/bureau_web/controllers/error_json_test.exs
* creating bureau/lib/bureau/repo.ex
* creating bureau/priv/repo/migrations/.formatter.exs
* creating bureau/priv/repo/seeds.exs
* creating bureau/test/support/data_case.ex
* creating bureau/lib/bureau_web/gettext.ex
* creating bureau/priv/gettext/en/LC_MESSAGES/errors.po
* creating bureau/priv/gettext/errors.pot
* creating bureau/priv/static/robots.txt
* creating bureau/priv/static/favicon.ico

Fetch and install dependencies? [Yn] Y
* running mix deps.get
* running mix deps.compile
```

## Dependencies for new API only service

If you open up the top-level mix.exs file you can see the dependencies that are setup on a brand new project:

```
  defp deps do
    [
      {:phoenix, "~> 1.7.20"},
      {:phoenix_ecto, "~> 4.5"},
      {:ecto_sql, "~> 3.10"},
      {:postgrex, ">= 0.0.0"},
      {:phoenix_live_dashboard, "~> 0.8.3"},
      {:telemetry_metrics, "~> 1.0"},
      {:telemetry_poller, "~> 1.0"},
      {:gettext, "~> 0.26"},
      {:jason, "~> 1.2"},
      {:dns_cluster, "~> 0.1.1"},
      {:bandit, "~> 1.5"}
    ]
  end
```

A couple of new things you might notice if you haven't done this in a while:

- the generation automatically included two telemetry related libraries: telemetry_metrics and telemetry_poller.
- instead of using cowboy as the HTTP server Phoenix now uses Bandit.
- there's a dependency on dns_cluster that is a fairly recent addition

Ecto and the Postgres library (postgrex), jason and phoenix should be very familiar to you. The phoenix_live_dashboard has also been included in the generated code for quite a while.

## aliases

The aliases function in the top level mix.exs file has been modified over time. It might look somewhat different than what you saw when you last generated it. But chances are it's pretty familiar looking.

```
  defp aliases do
    [
      setup: ["deps.get", "ecto.setup"],
      "ecto.setup": ["ecto.create", "ecto.migrate", "run priv/repo/seeds.exs"],
      "ecto.reset": ["ecto.drop", "ecto.setup"],
      test: ["ecto.create --quiet", "ecto.migrate --quiet", "test"]
    ]
  end
```

## Checking Dependencies

One of the first things I'd advise you to do it run `mix hex.outdated` on the project. This ensures that the template that was used to generate the Phoenix project used the latest versions of each of the libraries.

```
 mix hex.outdated
Dependency              Current  Latest  Status
bandit                  1.6.7    1.6.7   Up-to-date
dns_cluster             0.1.3    0.1.3   Up-to-date
ecto_sql                3.12.1   3.12.1  Up-to-date
gettext                 0.26.2   0.26.2  Up-to-date
jason                   1.4.4    1.4.4   Up-to-date
phoenix                 1.7.20   1.7.20  Up-to-date
phoenix_ecto            4.6.3    4.6.3   Up-to-date
phoenix_live_dashboard  0.8.6    0.8.6   Up-to-date
postgrex                0.20.0   0.20.0  Up-to-date
telemetry_metrics       1.1.0    1.1.0   Up-to-date
telemetry_poller        1.1.0    1.1.0   Up-to-date
```

So, as of today, this looks good. All the dependencies are "Up-to-date".

## Examining the New Things

The "new" things I noted are:

- the generation automatically included two telemetry related libraries: telemetry_metrics and telemetry_poller.
- instead of using cowboy as the HTTP server Phoenix now uses Bandit.
- there's a dependency on dns_cluster that is a fairly recent addition

Let's examine how those are being used and think about whether we need to modify any existing apps that we have to include the functionality that they bring.

## telemetry_metrics and telemetry_poller

These are libraries that you may already be using. Perhaps you are using something that wraps them instead.

At the lowest level (that you're apt to deal with as an Elixir developer) is the telemetry library. This library allows code that you write as part of an application and libraries that you use to generate events. It provides a common means of communicating measurement information.

The telemetry_metrics library is written in Elixir. The telemetry_metrics library "provides a common interface for defining metrics based on :telemetry events. These metrics can then be published to different backends using our Reporters API". This is the key thing for the library. It tries to normalize telemetry events by allowing the developer to describe what they are (counter? last_value?) and the expectations for a Reporter that will send those events on to an external system. Note that no external system is required. The library has a `Telemetry.Metrics.ConsoleReporter` that is a useful way of debugging the information provided (measurements and metadata) when events are generated.

The telemetry_poller library is written in Erlang. It allows an app to "periodically collect measurements and dispatch them as Telemetry events". The library itself starts a telemetry_poller to periodically gather VM statistics (memory used, processes, run queues, etc) and send them to the telemetry system. The app can define its own telemetry_poller with its own events of interest. The library documentation describes how this is done.

The reason to have a telemetry poller is that there is no "triggering event" to generate the metric, A poller is needed to gather the metrics.

If you look at the generated code in the web directory you'll see there is a file called "telemetry.ex".

```
defmodule BureauWeb.Telemetry do
  use Supervisor
  import Telemetry.Metrics

  def start_link(arg) do
    Supervisor.start_link(__MODULE__, arg, name: __MODULE__)
  end

  @impl true
  def init(_arg) do
    children = [
      # Telemetry poller will execute the given period measurements
      # every 10_000ms. Learn more here: https://hexdocs.pm/telemetry_metrics
      {:telemetry_poller, measurements: periodic_measurements(), period: 10_000}
      # Add reporters as children of your supervision tree.
      # {Telemetry.Metrics.ConsoleReporter, metrics: metrics()}
    ]

    Supervisor.init(children, strategy: :one_for_one)
  end

  def metrics do
    [
      # Phoenix Metrics
      summary("phoenix.endpoint.start.system_time",
        unit: {:native, :millisecond}
      ),
      summary("phoenix.endpoint.stop.duration",
        unit: {:native, :millisecond}
      ),
      summary("phoenix.router_dispatch.start.system_time",
        tags: [:route],
        unit: {:native, :millisecond}
      ),
      summary("phoenix.router_dispatch.exception.duration",
        tags: [:route],
        unit: {:native, :millisecond}
      ),
      summary("phoenix.router_dispatch.stop.duration",
        tags: [:route],
        unit: {:native, :millisecond}
      ),
      summary("phoenix.socket_connected.duration",
        unit: {:native, :millisecond}
      ),
      sum("phoenix.socket_drain.count"),
      summary("phoenix.channel_joined.duration",
        unit: {:native, :millisecond}
      ),
      summary("phoenix.channel_handled_in.duration",
        tags: [:event],
        unit: {:native, :millisecond}
      ),

      # Database Metrics
      summary("bureau.repo.query.total_time",
        unit: {:native, :millisecond},
        description: "The sum of the other measurements"
      ),
      summary("bureau.repo.query.decode_time",
        unit: {:native, :millisecond},
        description: "The time spent decoding the data received from the database"
      ),
      summary("bureau.repo.query.query_time",
        unit: {:native, :millisecond},
        description: "The time spent executing the query"
      ),
      summary("bureau.repo.query.queue_time",
        unit: {:native, :millisecond},
        description: "The time spent waiting for a database connection"
      ),
      summary("bureau.repo.query.idle_time",
        unit: {:native, :millisecond},
        description:
          "The time the connection spent waiting before being checked out for the query"
      ),

      # VM Metrics
      summary("vm.memory.total", unit: {:byte, :kilobyte}),
      summary("vm.total_run_queue_lengths.total"),
      summary("vm.total_run_queue_lengths.cpu"),
      summary("vm.total_run_queue_lengths.io")
    ]
  end

  defp periodic_measurements do
    [
      # A module, function and arguments to be invoked periodically.
      # This function must call :telemetry.execute/3 and a metric must be added above.
      # {BureauWeb, :count_users, []}
    ]
  end
end
```

BureauWeb.Telemetry is a Supervisor. It starts the GenServer `:telemetry_poller` that is in the telemetry_poller library. That GenServer gets passed whatever metrics are returned by the `periodic_measurements/0` function. By default this is an empty list. So no periodic measurements are reported. The process is started and does nothing.

A second GenServer is commented out:

```
{Telemetry.Metrics.ConsoleReporter, metrics: metrics()}
```

This used a module in the telemetry library to report the metrics returned by the `metrics/0` function to the console.

See the post (Ecto Telemetry)[2024-09-02-ecto-telemetry.md] for more information.

## bandit

If you've programmed with Phoenix for a while then you are used to seeing the cowboy library in the list of dependencies. Bandit's HTTP/1.x engine is up to 4x faster than Cowboy depending on the number of concurrent requests. When comparing HTTP/2 performance, Bandit is up to 1.5x faster than Cowboy. Work was done to make the bandit library a drop-in replacement for cowboy. For a period of time you had to specify you wanted to use bandit when generating a Phoenix app. At this point bandit is the default.

Bandit is written in Elixir. It's a good project to look at because the code is clean and can be understood by app developers. This was not the case with cowboy.

## dns_cluster

This was introduced to Elixir/Phoenix to allow Phoenix apps to setup DNS clustering out of the box. If you aren't using this you can remove it from your project.

For more information on this look up Chris McCord's Elixir Conference 2023 talk.

## See the telemetry working

If you want to see telemetry info in your app you can uncomment out the line `{Telemetry.Metrics.ConsoleReporter, metrics: metrics()}` in telemetry.ex. Then when you run your app locally you'll see something like this:

```
[Telemetry.Metrics.ConsoleReporter] Got new event!
Event name: vm.memory
All measurements: %{atom: 909553, atom_used: 885163, binary: 4214448, code: 15564055, ets: 1510568, processes: 21976920, processes_used: 21968208, system: 56001457, total: 77978377}
All metadata: %{}

Metric measurement: :total [via #Function<5.16980591/1 in Telemetry.Metrics.maybe_convert_measurement/2>] (summary)
With value: 77978.37700000001 kilobyte
Tag values: %{}

[Telemetry.Metrics.ConsoleReporter] Got new event!
Event name: vm.total_run_queue_lengths
All measurements: %{cpu: 1, io: 0, total: 1}
All metadata: %{}

Metric measurement: :total (summary)
With value: 1
Tag values: %{}

Metric measurement: :cpu (summary)
With value: 1
Tag values: %{}

Metric measurement: :io (summary)
With value: 0
Tag values: %{}
```

## Summary

Periodically get the latest phx.new and generate a new app to see changes that have occurred since you ran it (possibly many years prior). See what you might want to change in your app to align with the current way of creating a Phoenix app.
