---
layout: post
title: Flushing Ecto db connections periodically in Elixir
date: 2024-05-29 15:05:00
description: Sample code to show how to use disconnect_all
categories: elixir
tags: ecto myxql
giscus_comments: true
---

In Elixir's db_connection library version 2.4.1 and above there is a
facility to disconnect all the connections.

The myxql module `MyXql.Client` `do_connect/1` function is what actually translates
your “hostname” into an IP address (via `gen_tcp.connect/4`). By periodically calling
`disconnect_all/3` all the existing connections are dropped (within the time limit
you specify) and `do_connect/1` gets called again for each connection so if there has
been a DNS change it’ll be picked up.

I wouldn't try this unless you can identify an obvious problem. The db_connection
library is managing some complex things already. You don't want to add complexity
to it if you can avoid it. In any case, here's some sample code that can do the
work.

```
defmodule Ecto.PeriodicFlush do
  @moduledoc """
  Run a background process to periodically disconnect from databases
  """
  use GenServer

  @flush_interval 30_000

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: opts[:name] || __MODULE__)
  end

  @doc """
  Flush all the connections for a list of Ecto.Repo's
  Flushing connections ensures that our driver does a DNS lookup
  again
  """
  def flush_connections(repos) when is_list(repos) do
    Enum.each(repos, fn repo ->
      %{pid: pid, opts: opts} = Ecto.Adapter.lookup_meta(repo)
      DBConnection.disconnect_all(pid, 15_000, opts)
    end)
  end

  @impl true
  def init(opts) do
    flush_interval = Keyword.get(opts, :flush_interval, @flush_interval)
    repos = Keyword.get(opts, :repos, [])
    schedule_flush(flush_interval)
    {:ok, %{flush_interval: flush_interval, repos: repos}}
  end

  @impl true
  def handle_info(:flush_connections, %{flush_interval: flush_interval, repos: repos} = state) do
    flush_connections(repos)
    schedule_flush(flush_interval)
    {:noreply, state}
  end

  defp schedule_flush(flush_interval) do
    Process.send_after(self(), :flush_connections, flush_interval))
  end
end
```
