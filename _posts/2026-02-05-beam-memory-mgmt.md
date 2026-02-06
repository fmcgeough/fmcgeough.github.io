---
layout: post
title: BEAM Memory Management
date: 2026-02-05
description: Using NotebookLM to study BEAM Memory Management
tags:
categories: elixir
giscus_comments: true
---

I've found NotebookLM an effective study tool. I wanted an infographic to describe the BEAM memory management and it did a decent job of it.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" 
        path="https://github.com/fmcgeough/blog_posts/blob/main/img/2026-02-05-beam-memory-management.png?raw=true" 
        class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    BEAM Memory Management
</div>

One of the reasons to like NotebookLM is that it allows the user to specify the sources. For the infographic above I used:

- "Erlang Documentation: Garbage Collection" https://www.erlang.org/doc/apps/erts/garbagecollection.html
- "How Elixir Lays Out Your Data In Memory" https://www.honeybadger.io/blog/elixir-memory-structure/
- "Erlang Documentation: Memory Usage" https://www.erlang.org/doc/system/memory.html
- "Stuff Goes Bad: Erlang in Anger" https://s3.us-east-2.amazonaws.com/ferd.erlang-in-anger/text.v1.1.0.pdf
- "Wikiwiki - Generational Garbage Collection" https://wiki.c2.com/?GenerationalGarbageCollection

I think its important for Elixir developers to wrap their heads around the BEAM's memory management. One problem I've seen (more than once) is for developers to write processes that 1) run for as long as a service itself is running; 2) use memory very actively on startup and then are largely inactive. The problem that occurs is that the memory allocated for the active period is moved to the "Old Heap" and then not collected because the process runs so few reductions after the initial burst.

So, how does this work? Erlang (and Elixir) each use a generational garbage collector. Each process has its own young and old heaps. Most terms are created on the "young" heap. If data survives a garbage collection cycle, it is promoted to the "old" heap. While the young heap is collected frequently, the old heap is only reclaimed during a "full sweep". In a long-running process, if the process enters a state where it receives fewer messages or performs less work, a significant amount of time may pass before another full sweep is triggered. Long-running processes that act as routers or middlemen are particularly prone to memory bloat because they acquire references to every binary that passes through them.

If you notice this happening with your code and want to ensure you are running with a smaller footprint you can trigger a "full sweep" in your GenServer. This can be done in two ways. When you return a tuple from a GenServer call you can include `:hibernate` in the tuple to trigger the "full sweep" or you can use the `:hibernate_after` option when starting the GenServer. If this option is present, the GenServer process awaits any message for the given number of milliseconds and if no message is received, the process goes into hibernation automatically (by calling `:proc_lib.hibernate/3`).

If you can rework your design to use short lived processes than this problem goes away. Short-lived processes avoid the performance and memory penalties of data being promoted to the old heap and waiting for a full sweep. When a process dies, its entire stack and heap are deallocated at once, and all reference counts for binaries it held are decremented immediately.
