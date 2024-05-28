---
layout: post
title: Debugging Elixir in Production
date: 2024-02-19 08:53:13
description: Some debugging suggestions specific to Elixir production
categories: Elixir
tags: Elixir
---

Assuming you are making good use of Logging and generating metrics you may still encounter
problems in an Elixir allocation that requires more than that in order to understand
what is going on and address the problem.

## observer_cli

You absolutely should include the [observer_cli
library](https://hex.pm/packages/observer_cli) as a dependency in your project. It's
a wonderful tool and it brings in some other tooling that I've found helpful.

What is observer_cli?

- Based on recon (by the ubiquitous Fred Hebert)
- It makes it (relatively) easy to view high memory consumption and find the process responsible
- Good for general identification of issues but may not lead you to the actual underlying cause
- Checkout https://hexdocs.pm/observer_cli/ for lots of (sort of awkwardly written) documentation about how to use this

The library provides a bare-bones interface that is full of information. To start it open up
a remote console:

```
iex> :observer_cli.start
```

The start function line shows that the code for observer_cli is written in Erlang.

Some of what's available in the text UI:

- Process Information
  - Number Reductions Run
  - Current Function Executing
  - Size of the Message Queue
- Memory Information
  - Total Allocated
  - Memory for ETS
  - Memory for Atoms used in your application
  - Memory used by your application's processes
- Network
  - Ports in / out
  - Number bytes transferred
  - Memory Used

There's a lot there! If you are in the midst of an incident or problem then you might want:

- sort by Reductions if you having an issue where the CPU usage is high.
  Reductions is kind of an archaic term but you can think of it as analogous to
  CPU usage.
- examine memory growth.
  - Are the number of atoms used by your application constantly increasing? This
    will ultimately cause a crash. You should search your code for cases where you
    are dynamically allocating atoms (especially as a result of input that you do
    not have control over).
  - Is ETS usage growing in an unexpected way?
  - Is the amount of memory used by your processes continually increasing? There may be
    a garbage collection issue that you must address.

## Rolling your own - Finding high CPU processes

The observer_cli UI provides the ability to find high CPU processes but you can do so yourself.

```
Process.list() |> Enum.map(fn pid ->
  info = :erlang.process_info(pid)
  %{
    reductions: info[:reductions],
    name: info[:registered_name],
    current_function: info[:current_function]
  }
end) |> Enum.sort(&(&1[:reductions] >= &2[:reductions])) |> Enum.take(5)
```

This finds the 5 processes that have had the most reductions (but it’s over life of the VM).

## Rolling your Own - get_state

The function `:sys.get_state/1` can be used to output a process' state.

You can use this to (safely) get the state that’s associated with a running
GenServer. The execution time is slower than if the GenServer itself provided a
handle_call to return its state (i.e. don’t put get_state calls in your regular
code ordinarily) but sometimes that’s not available (or its a library’s
GenServer that you have no control over). If you need to examine a GenServer’s
internal state then this is what you use.

Note: if a process is really messed up calling `:sys.get_state/1` can timeout.

## Rolling your own - Erlang memory

The function `:erlang.memory/0` gives a quick overview of the current state of memory in
your application.

```
iex> :erlang.memory
[
  total: 36168216,
  processes: 15933632,
  processes_used: 15932480,
  system: 20234584,
  atom: 442553,
  atom_used: 424872,
  binary: 896224,
  code: 8184066,
  ets: 592096
]
```

This same information is available in observer_cli. The key/values returned are:

- total - The total amount of memory currently allocated. This is the same as
  the sum of the memory size for processes and system.
- processes - The total amount of memory currently allocated for the Erlang
  processes.
- processes_used - The total amount of memory currently used by the Erlang
  processes. This is part of the memory presented as processes memory.
- system - The total amount of memory currently allocated for the VM that is not
  directly related to any Erlang process.
- atom - The total amount of memory currently allocated for atoms. Makes up part
  of system value.
- atom_used - The total amount of memory currently used for atoms.
- binary - The total amount of memory currently allocated for binaries. Makes up
  part of system value.
- code - The total amount of memory currently allocated for Erlang/Elixir code.
  Makes up part of system value.
- ets - The total amount of memory currently allocated for ETS tables. Makes up
  part of system value.

You can define a simple function to examine memory over a period of time. For example:

```
iex> analyze_mem = fn(wait_ms) ->
  memory1 = :erlang.memory();
  Process.sleep(wait_ms);
  memory2 = :erlang.memory();
  Enum.map(memory1, fn {key, value1} -> value2 = Keyword.get(memory2, key); {key, value1 - value2} end) |> Map.new() |> Map.put(:wait_ms, "Memory changes after #{wait_ms} milliseconds")
end
```

You can use this function to show a memory change over a period of time of
interest. For example the call belows shows the memory change after 250
millseconds:

```
iex> analyze_mem.(250)
 analyze_mem.(250)
%{
  atom: 0,
  atom_used: 0,
  binary: 296,
  code: 0,
  ets: 0,
  processes: -258080,
  processes_used: -258080,
  system: 296,
  total: -257784,
  wait_ms: "Memory changes after 250 milliseconds"
}
```

## Tracing Function Calls

There are a few approaches built in to debug function calls in production. The best one
I've found is `recon`. As noted above, the observer_cli library is built on recon. This
means that if you include `observer_cli` as a dependency you get recon. Here's some
general information on it:

- Written by Fred Hebert and wraps :erlang.trace
- Production safe and basis for observer_cli. Has more than tracing
- Recon tracer links to the shell
- Which means...it shuts itself down on shell disconnect (unlike some other
  tracing which will run forever unless we explicitly stop it)
- So even if you had network interruption and got kicked out of your session
  your tracing is going to be turned off
- Limits total amount of trace info (unlike some more primitive tracing which will output everything….forever)

Here's an example of a simple trace:

```
:recon_trace.calls({Map, :get, :return_trace}, 2)
```

In the first argument (the tuple `{Map, :get, :return_trace}`) its saying "I
want to trace any calls to Map.get. Notice that we don't specify the arity.
Either a call to `Map.get/2` or `Map.get/3` will be traced. The `:return_trace`
says "I want to get the return value for the function call". the second
parameter is the number of times to trace. A trace of either the call to the
function or the return value counts as a trace. So this is saying "I want to
see what is passed to Map.get and then I want to see the return value".

```
iex> a_map = %{a: 133, bb: 3243, ccc: 99221}
iex> :recon_trace.calls({Map, :get, :return_trace}, 2)
iex> Map.get(a_map, :bb)
ex(3)> Map.get(a_map, :bb)
3243

16:22:52.750722 <0.1472.0> 'Elixir.Map':get(#{a=>133, bb=>3243, ccc=>99221}, bb)

16:22:52.756306 <0.1472.0> 'Elixir.Map':get/2 --> 3243
Recon tracer rate limit tripped.
```

Recon tracing also provides match spec pattern matching. This is a obtuse skill and
is explained better in the doc then I could do. Let's assume we want to do the same
thing but this time only trace if the key passed in to `Map.get` is `:ccc`. We could
do this:

```
iex> match = [{[:_, :"$1"], [{:==, :"$1", :ccc}], [{:return_trace}]}]
[{[:_, :"$1"], [{:==, :"$1", :ccc}], [{:return_trace}]}]
iex>  :recon_trace.calls({Map, :get, match}, 2)
2
iex(6)> Map.get(a_map, :a)
133
iex(7)> Map.get(a_map, :bb)
3243
iex(8)> Map.get(a_map, :ccc)
99221

16:28:03.217740 <0.1472.0> 'Elixir.Map':get(#{a=>133, bb=>3243, ccc=>99221}, ccc)

16:28:03.217830 <0.1472.0> 'Elixir.Map':get/2 --> 99221
Recon tracer rate limit tripped.
```

Notice that that the trace did not occur until the pattern was matched. Other calls were
not traced.

Note: you can only trace public functions.

You can also trace functions processes.

```
pid = Process.whereis(MyProcess)
:recon_trace.calls({MyProcess, :handle_call, :_}, 10, [{pid, :all}])
```

This is saying we want to trace any `handle_call` functions for a particular
process (identified by `pid`).

## Other Helpful Things in recon

If you have a `pid` for a process you are interested in you can use recon to get
information about it.

```
iex> pid = Process.whereis(MyProcess)
iex> :recon.info(pid)
[
  meta: [
    registered_name: MyProcess,
    dictionary: [
      "$ancestors": [MyProcess.Supervisor, #PID<0.708.0>],
      "$initial_call": {MyProcess, :init, 1}
    ],
    group_leader: #PID<0.707.0>,
    status: :waiting
  ],
  signals: [
    links: [#PID<0.709.0>],
    monitors: [],
    monitored_by: [],
    trap_exit: false
  ],
  location: [
    initial_call: {:proc_lib, :init_p, 5},
    current_stacktrace: [
      {:gen_server, :loop, 7, [file: 'gen_server.erl', line: 394]},
      {:proc_lib, :init_p_do_apply, 3, [file: 'proc_lib.erl', line: 249]}
    ]
  ],etc. etc.
```

## References

- [Erlang :sys module](https://erlang.org/doc/man/sys.html)
- [Erlang :trace](http://erlang.org/doc/man/erlang.html#trace-3)
- [Erlang dbg pattern matching](https://erlang.org/doc/man/dbg.html#fun2ms-1)
- [ex2ms - generate pattern matches for Elixir functions](https://hex.pm/packages/ex2ms)
- [observer_cli](https://github.com/zhongwencool/observer_cli)
- [recon - what observer_cli is based on](https://hex.pm/packages/recon)
- [Erlang in Anger](https://s3.us-east-2.amazonaws.com/ferd.erlang-in-anger/text.v1.1.0.pdf)
