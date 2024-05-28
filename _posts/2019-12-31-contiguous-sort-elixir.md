---
layout: post
title: Elixir, Programming Puzzles and Sorting Arrays
date: 2019-12-31 07:53:13
description: Elixir puzzles
categories: Elixir
tags: Elixir
---

There are a number of programming puzzle sites that are worthwhile tools for learning
a language, brushing up on your skills or just exploring how other people might solve
problems differently than you. I think this is all good. I've found it especially helpful
in learning a new language. One of the interesting things about trying to solve some of
these puzzles with a language like Elixir is that you have to work through the problem
in a different way then you would in a language like "C" or Java. Arrays and sorting are
common problems where you'd index into an Array with those languages but you want to avoid
that in Elixir since you're dealing with Lists and iterating into a List gets expensive.
Here's an example of a problem where I solved it a different way.

Say you're given the problem of determining if it's possible to sort an array of
integers by reversing one of the array's contiguous subarrays. So, for example,
if you are given the input: `[-1, 5, 4, 3, 2, 8]` you can reverse the subarray
`[5, 4, 3, 2]` and the entire array would end up sorted. But, for the input
`[-1, 5, 4, 3, 2, -5]` there is no subarray that can be reversed and end up with
a sorted array. The output from your function should just be true (a subarray
exists that can be reversed to provide sorted array) or false (no such subarray exists).
Additional constraints are that if there are any duplicates in the input then
the function should return false.

If you take a step back from the problem then you realize that in order for the function
to return true the input has to consist of a subarray of 0..n ascending elements, then
a subarray of 0..n descending elements (where the lowest number of the subarray is > then
the highest number of the first subarray) and then a subarray of 0..n ascending elements
again (where the lowest number of the last subarray is > the highest number in the descending
subarray).

In the example, `[-1, 5, 4, 3, 2, 8]` the subarrays that need to be identified are
`[-1]` and `[5, 4, 3, 2]` and `[8]`. Since -1 is < 2 (the last - lowest - element in
subarray2) and since 5 is < 8 (5 is the highest element in subarray 2) then the function
should respond "true".

Okay, its hopefully clear that those are the 3 subarrays for this input but what questions
do we need to answer to figure out whether reversing that 2nd subarray will leave the
array sorted? Starting from the first couple elements of the list [-1, 5] we know that
5 > -1 so 1) we know that we have an ascending subarray consisting of at least -1 and
possibly 5. But whether 5 is part of the ascending subarray depends on the next element.
Since the next element is 4 it means that 5 is the first element in our descending subarray.
We'd keep traversing elements in the list after 4 until we find one that is greater than
the element that proceeded it. That's how we get to [8] as the last subarray.

Now, since the 2nd subarray was in descending order and since the 3rd subarray ascends
(and stops in this case) with 8 and 8 is greater than our first element in the descending
subarray we "know" that this array can be sorted by reversing that descending subarray.

The data that it appears that we need to proceed thru the list and make a determination
is: 1) what's the current status (:leading_ascending, :descending, :trailing_ascending); 2) what's the maximum (last) number in subarray1 (this allows us to check for whether any
number in the 2nd subarray is less than max_subarray1); 3) what's the maximum (first) number in
subarray2 (this allows us to check for whether the number that starts the 3rd ascending
subarray is less than max_subarray2). Since we may or may not have an initial ascending
section the value of max_subarray1 can be nil.

If we put all the logic into our state module then the driver to determine true/false
becomes pretty simple:

```
defmodule ContiguousSubArray do
  def reverse_to_sort([num | t]) do
    reverse_to_sort(t, num, ReverseSortState.new(num))
  end

  def reverse_to_sort([num | t], prev, state) do
    IO.puts("#{inspect(state)}")

    case ReverseSortState.advance(state, num, prev) do
      false -> false
      new_state -> reverse_to_sort(t, num, new_state)
    end
  end

  def reverse_to_sort([], _, _state), do: true
end
```

The ContiguousSubArray creates a new ReverseSortState and then calls into the function
`reverse_to_sort/3`. In that function we call `ReverseSortState.advance/3` and pass the
current state of our analysis of the input, the value in the array we're currently on,
and the previous value in the array. If the `advance` function returns a state then all
is still good and we move forward one position by calling reverse_to_sort/3 recursively.
If `advance' returns false then the input didn't meet our expectations and we return false
to the caller. Our ReverseSortState is:

```
defmodule ReverseSortState do
  defstruct [:max_subarray2, :max_subarray1, :status]

  def new(num) do
    %__MODULE__{
      max_subarray2: num,
      max_subarray1: nil,
      status: :leading_ascending
    }
  end

  def advance(%{status: status} = state, num, prev) when status == :leading_ascending do
    cond do
      num > prev ->
        %{state | max_subarray2: num, max_subarray1: prev}

      num == prev ->
        false

      num < prev and num_greater_than_max_subarray1?(state, num) ->
        %{state | status: :descending}

      true ->
        false
    end
  end

  def advance(%{status: status} = state, num, prev) when status == :descending do
    cond do
      num < prev and num < state.max_subarray2 and num_greater_than_max_subarray1?(state, num) ->
        state

      num == prev ->
        false

      num > prev and num_greater_than_max_subarray1?(state, num) and num > state.max_subarray2 ->
        %{state | status: :trailing_ascending}

      true ->
        false
    end
  end

  def advance(%{status: status} = state, num, prev) when status == :trailing_ascending do
    case num > prev do
      true -> state
      false -> false
    end
  end

  defp num_greater_than_max_subarray1?(state, num) do
    is_nil(state.max_subarray1) or num > state.max_subarray1
  end
end
```

We can run this code thru some unit tests with:

```
defmodule ContiguousSubArrayTest do
  @inputs [
    {[-1, 5, 4, 3, 2, 8], true},
    {[1, 3, 2, 5, 4, 6], false},
    {[2, 3, 2, 4], false},
    {[19, 32, 23], true},
    {[5, 4, 3, 2, 1], true}
  ]

  def test do
    @inputs
    |> Enum.map(fn {input, result} ->
      ContiguousSubArray.reverse_to_sort(input) == result
    end)
  end
end
```

On puzzles in general, there are a number of companies that use puzzles from these sites
(or some developed internally) as filters for hiring. This is a bit frustrating if you
are applying for a job working on what amounts to a CRUD app but are asked to solve puzzles
that would never come up in the actual work for the job. But its not uncommon. If you
know the company you'd like to work at will throw these type of puzzles at you then I'd
recommend spending some time on one or more of the programming puzzle sites and work
thru some puzzles. If you can find out which site the company uses then practice on that
since each has their own UI and idiosyncrasies and you want to be familiar with it
before you start working on puzzles as part of interview (ordinarily as a timed exercise).

Sites to check out that have Elixir support:

- [https://codesignal.com/](https://codesignal.com/)
- [https://exercism.io/](https://exercism.io/)
