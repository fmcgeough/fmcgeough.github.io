---
layout: post
title: Self Describing Numbers
date: 2025-10-01
description: Using Elixir to find self describing numbers
tags:
categories: elixir
giscus_comments: true
---

I like the Elixir language. One challenging aspect of liking a programming language that is not used to the extent of Javascript, Java, or Go is that it may be a bit tougher to find examples. So I figured I'd periodically post some Elixir implementations of algorithms. Sites like exercism.io are also wonderful sources to explore using Elixir for simple algorithm based tasks.

This post is on self describing numbers. From Wikipedia: A self-describing number is an integer in a given base where each digit's position indicates the quantity of that digit within the number. For example, in base 10, the number 2020 is a self-describing number because its first digit (2) indicates the two zeros in the number, the second digit (0) indicates no ones, the third digit (2) indicates two twos, and the fourth digit (0) indicates no threes.

For example, in base 10, the number 6210001000 is self-descriptive for the following reasons:

In base 10, the number has 10 digits, indicating its base;

- It contains 6 at position 0, indicating that there are six 0s in 6210001000
- It contains 2 at position 1, indicating that there are two 1s in 6210001000
- It contains 1 at position 2, indicating that there is one 2 in 6210001000
- It contains 0 at position 3, indicating that there is no 3 in 6210001000
- It contains 0 at position 4, indicating that there is no 4 in 6210001000
- It contains 0 at position 5, indicating that there is no 5 in 6210001000
- It contains 1 at position 6, indicating that there is one 6 in 6210001000
- It contains 0 at position 7, indicating that there is no 7 in 6210001000
- It contains 0 at position 8, indicating that there is no 8 in 6210001000
- It contains 0 at position 9, indicating that there is no 9 in 6210001000

Elixir algorithm for determining whether an integer is self descriptive for base 10.

```
defmodule IntegerAnalyzer do
    def self_descriptive?(n) when is_integer(n) and n >= 0 do
        {val_with_index, freq} = analyze(n)
        self_descriptive_check(val_with_index, freq)
    end

    defp analyze(n) do
        val = n |> Integer.to_charlist() |> Enum.map(fn x -> x - 48 end)
        {Enum.with_index(val), Enum.frequencies(val)}
    end

    defp self_descriptive_check([], _freq), do: true
    defp self_descriptive_check([{occurrences, pos}|t], freq) do
        if Map.get(freq, pos, 0) == occurrences do
            self_descriptive_check(t, freq)
        else
            false
        end
    end
end
```

This is not particularly fast but for a few million values it's okay. On my Macbook Pro it takes about a second and a half.

```
iex> 1..10_000_000 |> Enum.filter(&IntegerAnalyzer.self_descriptive?(&1))
[1210, 2020, 21200, 3211000]
```

The self_descriptive? function takes the integer and builds a charlist. It then maps that list to values by subtracting the ASCII value for zero (48). The next
line builds a tuple consisting of the list with the number of occurrences and the value along with the frequencies for the charlist. The frequencies shows how many times an element occurs in a list. Here's a short demonstration of what the code does:

```
> val = 24234900 |> Integer.to_charlist() |> Enum.map(fn x -> x - 48 end)
[2, 4, 2, 3, 4, 9, 0, 0]
> Enum.with_index(val)
[{2, 0}, {4, 1}, {2, 2}, {3, 3}, {4, 4}, {9, 5}, {0, 6}, {0, 7}]
> Enum.frequencies(val)
%{0 => 2, 2 => 2, 3 => 1, 4 => 2, 9 => 1}
```

The self_descriptive_check is a recursive function that returns true if it gets to the end of the list without a problem and false if it ever finds that the number of occurrences in the frequency for a position doesn't match the value built with the Enum.with_index.
