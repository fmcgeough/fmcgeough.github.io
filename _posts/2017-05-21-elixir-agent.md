---
layout: post
title: 'Elixir, Phoenix and Using Agent for State'
categories:
- Elixir
tags: []
---
One of the real joys of Elixir is how easy it makes solving problems that
are relatively difficult or tedious in other languages. A lot of this comes
via wrapping the facilities in the underlying Erlang language and [OTP](http://learnyousomeerlang.com/what-is-otp).
One of the areas that I've just had an opportunity to explore are GenServer and Agent.

The original Phoenix web app that I wrote to explore Phoenix and Elixir is
a read-only administrative tool for my company. It extracts information out
of a relational db (Postgres) and Document db (Mongo) and provides an "under the
covers" view of what is being shown in both our end-user and admin UI's. This
allows support and operations folks to more quickly diagnose customer
configuration issues that are perhaps not at all obvious using our regular UI.

The ordinary user interaction with this app is that a customer calls with a
support request and the support or operations person turns to this Phoenix
app to look at the inner details of the customer's configuration. This usually
results in them being able to quickly pick out the problem.

So I met my initial goal of providing really quick views of various information
about customer's configurations but I recently finally got around to my other
goal which was to proactively discover issues in customer's configurations. I
didn't want to wait until a customer reported an issue. I wanted to do scans
of the configuration data across all customers and provide pages that displayed
common issues in configuration. Using [GenServer](https://hexdocs.pm/elixir/GenServer.html)
and [Agent](http://elixir-lang.org/getting-started/mix-otp/agent.html) I was able to quickly
put together a few pages of the most common configuration issues.

A GenServer worked ideally for my background task to check on configuration
issues. To create this GenServer I implemented start_link, init and handle_info.
On init I immediately start the work of checking customer's configuration info
and once that work is done I setup a scheduled call to perform the work again
periodically. This ended up looking something like this:

    def init(state) do
      schedule_work_immediately()
      {:ok, state}
    end

    def handle_info(:work, state) do
      Logger.info "Performing background work to analyze issues"
      do_work()
      schedule_work()
      {:noreply, state}
    end

    defp schedule_work_immediately() do
      Process.send_after(self(), :work, 10 * 1000) # Start in 10 seconds
    end

    defp schedule_work() do
      Process.send_after(self(), :work, 10 * 60 * 1000) # Every 10 minutes
    end

Although I could have stored the issues found when scanning inside the
GenServer itself it made more sense to me to store the data separately in
an Agent. This means that if the GenServer crashes for some reason the data
will be maintained. Since the Agent's only responsibility is to store and
provide access to the information, it's much less likely to run into
issues. It required very little coding to implement the Agent functionality
I needed. Like so little that after I finished typing in what I thought I needed I was
not sure at first whether I'd missed something or not. Here is the complete
implementation.

    defmodule SystemDetective.BackgroundStorage do
      require Logger

      def start_link do
        Logger.info "Starting our background storage Agent"
        Agent.start_link(fn -> %{} end, name: __MODULE__)
      end

      def get(key) do
        Agent.get(__MODULE__, fn state ->
          Map.get(state, key)
        end)
      end

      def put(key, data) do
        Agent.update(__MODULE__, fn(state) ->
          Map.put(state, key, %{time_updated: :calendar.local_time(), data: data }) end)
      end
    end

So, what happens is that the GenServer will call into SystemDetective.BackgroundStorage
to store issues under a particular key. The issues consist of a List of struct info
that describes the account, the particular component that is having an issue and some
info on how to address the issue. Each run of the GenServer will completely replace the
previous information so there was no added complexity of having to figure out how to
update state. When the GenServer puts some new information the function adds the
timestamp that the information was added. This allows the web page to display the
time that the data was last updated.

There is no storage of this data across restarts of the web app. This was not that
important since the information is refreshed every 10 minutes anyway. There is also no
queue of work with more than one GenServer since currently the work completely fairly
rapidly and there was no need to introduce that small amount of additional complexity.

A web page (controller, template, view) per problem area is associated with each of the
problems that this background task finds. A typical controller is almost no code as well:

    def index(conn, _params_) do
      args = BackgroundStorage.get(:abandoned_devices) |> build_args()
      render(conn, "index.html", args)
    end

    def build_args(nil) do
      [time_updated: nil, devices: []]
    end

    def build_args(%{time_updated: update_time, data: data}) do
      [devices: data, time_updated: update_time]
    end

The Phoenix templates "knows" that if the time_updated is nil then the background job
has not yet run.

Its admittedly a trivial example of using GenServer and Agent but what was extremely
nice about implementing the solution is that the code itself was trivial and aligned
so nicely with what I thought of as a trivial problem. In other languages and
frameworks it could get quite a bit more complicated.
