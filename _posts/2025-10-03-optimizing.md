---
layout: post
title: Code Optimization
date: 2025-10-07
description: Improving speed of self describing numbers
tags:
categories: elixir
giscus_comments: true
---

## In the Past

I was lucky that my first software job was working at a small boutique company that built full systems (hardware and software) on a contract basis for larger companies who lacked the expertise to build the product that they wanted to sell.

The CEO of the boutique company was a smart man who always wanted us to deliver the best possible product for the company that contracted us. There was a healthy atmosphere of reexamining whatever got built to see if 1) it met the customer requirements; 2) performance or user experience improvements might be possible. Meeting the customer requirements was required. Performance improvements were welcomed (and celebrated) as long as they wouldn't risk violation of our hard deliverable date. In other words, we couldn't endlessly tinker with what we were building.

## Lessons Learned

The practice of reexamining code and looking for performance improvements led to a fundamental (humbling) principle: "Any code, whether written by you or someone else, can have it's performance improved". You will hear throughout your career that some important code cannot possibly be made faster. This is almost always false.

There are some common approaches to improving performance.

- Look for code that can be moved out of a loop. This can be an easy win. If some variable is being initialized to the same value every time through a loop it should not be in the loop
- Examine the algorithm that is being used and what is actually needed to solve the problem. Leverage algorithm research. There are advances and they may not have been available at the time the code was originally written
- Examine the options within the language or tool you are using. Many (most) times code is written a certain way because it's just always been written that way. Or it was written that way because the tool used had no other option at the time. Is there a different way to approach solving the problem?
- Can the problem be worked on in parallel?

There are times where it's not worth the engineering time to work on slower performing code. You know its slow. You can have a clear idea of how you might make it faster. That doesn't necessarily mean that you should modify the code. Generally, if an engineer wanted to put in a performance improvement I'd consider how close we are to having to deliver and what the cost is to retest code if a change is made to "working code". If there's time it's probably worth creating a branch with the changes and a proof that it is faster (if nothing else).

## Measuring Performance with Elixir

If you're trying to make performance improvements you need to measure. It's odd to say that because it seems self-evident. However, I've worked places where folks would say "I made that code 3x faster" without actually having hard data to back it up. This is concerning not because it might be inaccurate but because it also may indicate that no testing was done on the improved code.

In any case, for whatever language you are using you need to know how to measure the passage of time. Since this blog is focused around Elixir I'll examine a couple of ways of measuring performance in Elixir.

### timer.tc

Elixir has access to the Erlang stdlib functions. There an available module (tc) that has a function (tc/1) that times a function. You pass a function into the `:timer` tc function and it returns a tuple where the first element is the total time in microseconds it took to run the function. You can take that value and divide by a million to get milliseconds for display.

Here's an example of timing an Elixir function using `:timer.tc/1`:

```
iex> func = fn -> 1..10_000 |> Enum.each(fn v -> v / 2 == 0  end) end
#Function<43.81571850/0 in :erl_eval.expr/6>
iex> func |> :timer.tc() |> elem(0) |> then(& &1/ 1_000_000)
0.0019
```

The timer tc function gives us milliseconds. The division shows the result as fractions of a second.

### system.monotonic_time

If we needed measurements in nanoseconds then we can use `:erlang.monotonic_time()`. This is what is used by the telemetry library used in Elixir to generate or process events. Here's an example of using `:erlang.monotonic_time()` using a timing module.

```
defmodule MonoTimer do
  def time(function) do
    start = :erlang.monotonic_time()
    result = function.()
    finish = :erlang.monotonic_time()

    duration_nano_seconds = :erlang.convert_time_unit(finish - start, :native, :nanosecond)

    {duration_nano_seconds, result}
  end
end

iex> func = fn -> 1..10_000 |> Enum.each(fn v -> v / 2 == 0  end) end
#Function<43.81571850/0 in :erl_eval.expr/6>
iex> MonoTimer.time(func)
{6433137, []}
```

Notice that in this case the function is returning both the time in nanoseconds it took to run the function and the result of the function run.

Both `:timer.tc/1` and `:erlang.monotonic_time/1` are functions that you'll probably use periodically to quickly measure a function's speed.

## Use a library

For production software you most likely need something more robust. For example, you may want to compare use of memory between two functions or you may have a function that really needs a bit of warmup before it is run to allow it's performance to come closer to what the real life usage is like. You might want to measure performance when the function is run within multiple processes simultaneously. For situations like this its best to use a library that has already been developed to meet these needs. The one that I use most often is benchee. It has almost 580,000 downloads reported in hex.pm as of October 4th, 2025.

## Using Benchee

In the blog post on [self describing numbers](https://fmcgeough.github.io/blog/2025/self-describing-numbers/) I had an Elixir algorithm that can determine whether a number is self-describing. It's slower than it could be. I'll use it to show a simple example of using benchee to improve Elixir performance.

The general algorithm used to identify a self descriptive number has two phases:

- First, extract the digits from the integer value into a list
- Second, build a map of the frequency of the occurrence of each number in the original integer where the key is the position in the list starting at 0 and the value is how many times that digit occurs, and then for each digit 0..n in the list look up the location in the frequencies built and check that the value returned matches the digit for that location

These two general steps seem required. We need to break out all the digits and then we need to do some form of analysis on the resulting list to see if it matches the criteria for a self-describing number. However, maybe one or both of these could be faster.

A first step is to create a new project `mix new self_describe`, add `benchee` as a dependency. I add the benchee library as `:dev` only so its only included on dev builds.

```
{:benchee, "~> 1.4", only: :dev}
```

Next I'll get the original algorithm into a module. Since we "know" that we might want to test alternatives to both extracting digits from an integer and performing the algorithm I'll modify the code from previous blog post to make that easier.

```
defmodule IntegerAnalyzer do
  @moduledoc """
  Provide analysis of integer(s)
  """

  @max_allowed_integer 9_999_999_999

  @doc """
  Determine if the integer provided as argument is self-descriptive?

  A self-describing number is an integer in a given base where each digit's position indicates the quantity of that digit within the number.
  """
  @spec self_descriptive?(integer) :: boolean
  def self_descriptive?(n) when is_integer(n) and n <= @max_allowed_integer and n > 0 do
    n |> digits_from_integer() |> self_descriptive_algorithm()
  end

  @doc """
  Convert the input integer into a list of its integer digits
  """
  @spec digits_from_integer(integer) :: [integer]
  def digits_from_integer(n) do
    n |> Integer.to_charlist() |> Enum.map(fn x -> x - 48 end)
  end

  defp self_descriptive_algorithm(val) do
    self_descriptive_check(Enum.with_index(val), Enum.frequencies(val))
  end

  defp self_descriptive_check([{occurrences, pos} | t], freq) do
    if Map.get(freq, pos, 0) == occurrences do
      self_descriptive_check(t, freq)
    else
      false
    end
  end
end
```

To use benchee we'll want a module that runs our test. In the test function we'll run 500,000 integers through the algorithm.

```
defmodule IntegerAnalyzerTester do
  @test_data 1_000_000_000..1_000_500_000

  def run_self_descriptive do
    Enum.filter(@test_data, &IntegerAnalyzer.self_descriptive?(&1))
  end

  def run_digits_from_integer_test do
    Enum.each(@test_data, &IntegerAnalyzer.digits_from_integer(&1))
  end
end
```

After starting up an iex session with `iex -S mix` we can use benchee to test the current performance. I'm going to cut out some of the benchee output to focus on the performance only.

```
iex> data = %{"original" => &IntegerAnalyzerTester.run_self_descriptive/0}
iex> Benchee.run(data); 0
Benchmarking self_descriptive? ...
Calculating statistics...
Formatting results...

Name               ips        average  deviation         median         99th %
original          7.65      130.68 ms     ±0.71%      130.68 ms      133.01 ms
```

The first argument to benchee run is a Map where the key is a user-friendly name and the value is the function to run. By default, benchee will run the function (over and over) for two seconds then it will test by running the function (over and over) for five seconds. The two seconds are a warmup for code that requires a warmup to get accurate results that reflect deployed software. Both of these durations are configurable.

The output from benchee includes two elements of interest:

- ips - iterations per second, aka how often can the given function be executed within one second (the higher the better - good for graphing), only for run times
- average - average execution time/memory usage (the lower the better)

We can use the first argument to pass in both the current implementation and an improved one with different user-friendly nanes. The benchee library outputs details on both individually and the difference between them.

So what improvements can we make?

## Improved Integer to Digits

It turns out there is no need for us to convert integers to digits. There is a function in the Integer module that does this for us. Since it's part of the core library chances are it does it faster than the original implementation. Here's a new function for the IntegerAnalyzer.

```
  @spec improved_digits_from_integer(integer) :: [integer]
  def improved_digits_from_integer(n) do
    n |> Integer.digits()
  end
```

## Avoiding Work When Its Not Needed

We can avoid the current algorithm if we can easily determine that a string of digits cannot possibly be self descriptive. It's fairly easy to realize that because each digit refers to the number of occurrences of its position number in the integer that if we added all the individual digits together and it was greater than the length of the digits list then we can't possibly have a self descriptive number. Here's the new function for the IntegerAnalyzer.

```
  # Return true if the sum of digits exceeds the length of the list because
  # this indicates the number cannot be self-descriptive
  defp sum_exceeds_len?(val) do
    Enum.sum(val) > length(val)
  end
```

## Create a Public Function for benchee

We'll need a IntegerAnalyzer function to pass into benchee to test our (hopefully) faster code. While I'm in there I'm going to add two functions to test the original digits from integer with the improved version.

```
  @spec improved_self_descriptive?(integer) :: boolean
  def improved_self_descriptive?(n) when is_integer(n) and n <= @max_allowed_integer do
    val = digits_from_integer(n)
    val |> sum_exceeds_len?() |> case do
      true -> false
      false -> self_descriptive_algorithm(val)
    end
  end
```

## Using benchee to compare

The final step is to modify our IntegerAnalyzerTester:

```
defmodule IntegerAnalyzerTester do
  @test_data 1_000_000..1_500_000

  def run_self_descriptive do
    Enum.filter(@test_data, &IntegerAnalyzer.self_descriptive?(&1))
  end

  def run_improved_self_descriptive do
    Enum.filter(@test_data, &IntegerAnalyzer.improved_self_descriptive?(&1))
  end

  def run_digits_from_integer do
    Enum.each(@test_data, &IntegerAnalyzer.digits_from_integer(&1))
  end

  def run_improved_digits_from_integer do
    Enum.each(@test_data, &IntegerAnalyzer.improved_digits_from_integer(&1))
  end
end
```

## Final version of IntegerAnalyzer

```
defmodule IntegerAnalyzer do
  @moduledoc """
  Provide analysis of integer(s)
  """

  @max_allowed_integer 9_999_999_999

  @doc """
  Determine if the integer provided as argument is self-descriptive?

  A self-describing number is an integer in a given base where each digit's position indicates the quantity of that digit within the number.
  """
  @spec self_descriptive?(integer) :: boolean
  def self_descriptive?(n) when is_integer(n) and n <= @max_allowed_integer and n > 0 do
    n |> digits_from_integer() |> self_descriptive_algorithm()
  end

  @spec improved_self_descriptive?(integer) :: boolean
  def improved_self_descriptive?(n) when is_integer(n) and n <= @max_allowed_integer and n > 0 do
    val = improved_digits_from_integer(n)

    val
    |> sum_exceeds_len?()
    |> case do
      true -> false
      false -> self_descriptive_algorithm(val)
    end
  end

  @doc """
  Convert the input integer into a list of its integer digits
  """
  @spec digits_from_integer(integer) :: [integer]
  def digits_from_integer(n) do
    n |> Integer.to_charlist() |> Enum.map(fn x -> x - 48 end)
  end

  def improved_digits_from_integer(n) do
    n |> Integer.digits()
  end

  @doc """
  Return true if the sum of digits exceeds the length of the list because
  this indicates the number cannot be self-descriptive
  """
  @spec sum_exceeds_len?([integer]) :: boolean()
  def sum_exceeds_len?(val) do
    Enum.sum(val) > length(val)
  end

  defp self_descriptive_algorithm(val) do
    self_descriptive_check(Enum.with_index(val), Enum.frequencies(val))
  end

  defp self_descriptive_check([{occurrences, pos} | t], freq) do
    if Map.get(freq, pos, 0) == occurrences do
      self_descriptive_check(t, freq)
    else
      false
    end
  end
end
```

## Retest to compare improved to original

With all that in place we can run the test:

```
$ iex -S mix
iex> Benchee.run(
    %{
      "original" => &IntegerAnalyzerTester.run_self_descriptive/0,
      "new" => &IntegerAnalyzerTester.run_improved_self_descriptive/0}
    )

Name               ips        average  deviation         median         99th %
new              38.88       25.72 ms     ±3.34%       25.90 ms       27.18 ms
original          7.99      125.08 ms     ±0.65%      125.11 ms      127.08 ms

Comparison:
new              38.88
original          7.99 - 4.86x slower +99.36 ms
```

It's easy to see the new approach gets much better performance. Close to 5x.

## Now what?

Well everything seems good. The improved code allows the function to run in 25 milliseconds vs 125 milliseconds in the original.

One useful thing to do when you see an improvement (especially a big one) is to work out why it's so much faster. In this case it'd be interesting to determine how many numbers we're actually rejected by the `sum_exceeds_len?/1` change. It's not too difficult to do with the reworked code. Out of the 500,000 integer
values let's see how many have a sum of digits that exceeds the length of the list.

```
iex> 1_000_000_000..1_000_500_000 |>
  Enum.map(&IntegerAnalyzer.improved_digits_from_integer/1) |>
  Enum.map(&IntegerAnalyzer.sum_exceeds_len?/1) |>
  Enum.filter(&(&1 == true)) |> Enum.count()
495205
```

That's a lot of numbers being rejected before running the more expensive algorithm. There's a cost with performing the sum and length. But it's not nearly as high as creating the frequency map and then looking up a value in the map.

The performance improvement looks okay. It's improved because its doing much less work.

## What about digits from integer?

How much speed improvement (or slowdown!) did we get by using the system solution `Integer.digits/1`?

```
Name               ips        average  deviation         median         99th %
new              87.89       11.38 ms     ±4.25%       11.25 ms       12.55 ms
original         53.81       18.58 ms     ±0.80%       18.58 ms       19.15 ms

Comparison:
new              87.89
original         53.81 - 1.63x slower +7.21 ms
```

Yep. It's faster. Not as dramatic as avoiding a bunch of unnecessary work but its faster and it's also built in to the Integer module. This means we're not maintaining code unnecessarily.

## Wrap up

This is a somewhat contrived example to demonstrate the basics of using benchee. There are additional related libraries that you can add that do things like format results as HTML and graph the results.
