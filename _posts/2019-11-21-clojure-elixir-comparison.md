---
layout: post
title: 'Clojure and Elixir'
categories:
- Clojure
- Elixir
tags: []
---
Here's some initial notes on coming to Clojure from Elixir. This isn't meant to be
a judgemental post. Its just my random thoughts on using a different model of functional
programming. My bias would obviously be towards the one I know (Elixir). So its worth
reading this with this background in mind. It'd be interesting to read someone's thoughts
on learning Elixir after using Clojure professionally. I've not found any blog posts about
that although I'm sure there are some out there.

I'll take a simple problem and explore some differences. I'll take an example problem right out of the ["Programming Clojure" book](https://pragprog.com/book/shcloj3/programming-clojure-third-edition) when its introducing some of the basic Clojure features. The problem is "take characters from a String up until the first consonant".

### Repl

Both languages have a repl. Repl's are regarded as "table stakes" with any modern language. Which
is kind of amusing historically since Lisp and Forth had repl's way back when and it didn't stop
folks from developing lots of languages that didn't have a repl and yet were very successful. But
times have changed. For developers who've come along in the last decade it might be a bit of a shock
to encounter a language without one.

Both languages have a concept of "take..while". In Clojure this is just `take-while`. Whereas in Elixir,
this function is inside the Enum module as `Enum.take_while`. The functionality is the same. Elements
are dropped from a list up until the the predicate/function evaluates to true. Here's how you'd go about
definining a solution to the problem in a repl in Clojure.

```
user=> (def vowel? #{\a\e\i\o\u})
#'user/vowel?
user=> (def consonant? (complement vowel?))
#'user/consonant?
user=> (take-while consonant? "the-quick-brown-fox")
(\t \h)
```

One thing about Clojure that is immediately different from Elixir is that you can define functions
like this in the repl. Although you can do this in Elixir its not ordinarily how you'd approach the
problem because functions are meant to live inside of modules in Elixir. You can define functions
without a module but the calling syntax ends up looking a bit odd. Here's the equivalent in Elixir.

```
iex> vowel? = &(&1 in ["a", "e", "i", "o", "u"])
#Function<6.127694169/1 in :erl_eval.expr/5>
iex> consonant? = &(not vowel?.(&1))
#Function<6.127694169/1 in :erl_eval.expr/5>
iex> "the-quick-brown-fox" |> String.graphemes() |> Enum.take_while(&(consonant?.(&1)))
["t" "h"]
```

The odd thing here (if you're new to Elixir) is that we're calling the function by using the `.`
operator. If the functions were in their own module it'd look like this:

```
iex> defmodule Letter do
...>   def vowel?(x), do: x in ["a", "e", "i", "o", "u"]
...>   def consonant?(x), do: not vowel?(x)
...> end
{:module, Letter,
 <<70, 79, 82, 49, 0, 0, 5, 104, 66, 69, 65, 77, 65, 116, 85, 56, 0, 0, 0, 150,
   0, 0, 0, 17, 12, 69, 108, 105, 120, 105, 114, 46, 80, 97, 114, 115, 101, 8,
   95, 95, 105, 110, 102, 111, 95, 95, 7, ...>>, {:consonant?, 1}}
iex> "the-quick-brown-fox" |> String.graphemes() |> Enum.take_while(&Letter.consonant?(&1))
["t", "h"]
```

### Namespaces

So, why can we define functions in the Clojure repl without any muss or fuss? It has to be do
with namespaces. When you start up a Clojure repl you'll be in a default namespace. The functions
you define in the repl become part of that namespace. The current namespace if you start up
a bare-bones Clojure repl is "user". Note: you can figure out what namespace you are in by using
`*ns*`:

```
user=> *ns*
#object[clojure.lang.Namespace 0x22581e62 "user"]
```

In Elixir all functions are expected to live in modules. And when you start iex (the Elixir repl)
you aren't "in a module". There's no default and it really doesn't make sense that there would be
given how Elixir and Erlang work. However, as I demonstrated you can define named
functions outside of a module. You just have to invoke them with the `.` operator and that's
ordinarily not what you'd see in Elixir code.

You can have multiple namespaces in Clojure. And you can have multiple modules in Elixir. You can
nest modules within Elixir - although you'd want to have a good reason to do so. You don't nest
namespaces in Clojure - although you can define multiple namespaces in a file - and, again, you'd
need a really good reason to do so.

The documentation for [Elixir module](https://hexdocs.pm/elixir/Kernel.html#defmodule/2) is pretty
thorough. Likewise, the [Clojure namespace](https://clojure.org/reference/namespaces) doc explains
what these are pretty well.

### Where did my functions come from?

In the Closure code we used the function take-while, which seemed to appear out of nowhere. How
did that work? [Clojure take-while](https://clojuredocs.org/clojure.core/take-while) is in the
`clojure.core` namespace. So when the repl started that must have already been loaded. There's
code you can use in the repl to list the loaded namespaces.

```
user=> (->> (all-ns)
  #_=>      (map ns-name)
  #_=>      (map name))
("clojure.spec.gen.alpha" "net.cgrand.regex.unicode" "nrepl.middleware.interruptible-eval" "reply.completion" "reply.eval-modes.shared" "clojure.stacktrace" "clojure.tools.cli" "leiningen.core.classpath" "leiningen.core.eval" "leiningen.core.pedantic" "clojure.uuid" "bultitude.core" "complete.core" "reply.eval-modes.nrepl" "net.cgrand.parsley.util" "clojure.main" "user" "leiningen.repl" "clojure.test" "dynapath.dynamic-classpath" "net.cgrand.parsley.fold" "clj-stacktrace.repl" "dynapath.util" "clojure.edn" "reply.conversions" "clojure.core.server" "cemerick.pomegranate" "clojure.java.io" "clojure.core.specs.alpha" "nrepl.server" "net.cgrand.parsley.stack" "clojure.core.protocols" "reply.exit" "clj-stacktrace.utils" "nrepl.middleware.session" "clojure.pprint" "reply.signals" "nrepl.middleware.caught" "nrepl.bencode" "nrepl.middleware.load-file" "reply.exports" "nrepl.version" "clj-stacktrace.core" "clojure.instant" "leiningen.trampoline" "net.cgrand.parsley.tree" "clojure.spec.alpha" "reply.parsing" "leiningen.core.utils" "leiningen.core.project" "clojure.set" "nrepl.transport" "cemerick.pomegranate.aether" "net.cgrand.parsley.lrplus" "nrepl.ack" "clojure.string" "reply.main" "clojure.java.browse" "classlojure.core" "reply.reader.simple-jline" "clojure.java.javadoc" "clojure.repl" "reply.eval-modes.standalone" "reply.reader.jline.completion" "reply.eval-state" "net.cgrand.parsley" "trptcolin.versioneer.core" "leiningen.core.main" "net.cgrand.sjacket.parser" "clojure.template" "reply.hacks.printing" "nrepl.misc" "clojure.java.shell" "leiningen.core.user" "nrepl.core" "net.cgrand.regex.charset" "clojure.core" "net.cgrand.sjacket" "net.cgrand.parsley.grammar" "nrepl.config" "net.cgrand.regex" "clojure.walk" "nrepl.middleware" "nrepl.middleware.print" "reply.initialization" "clojure.zip" "reply.eval-modes.standalone.concurrency" "dynapath.defaults")
```

Wow! That's alot. But the most important one in this big list is
[clojure.core](https://clojuredocs.org/clojure.core). As noted in the Clojure
docs `clojure.core` "provides the bulk of the functionality you'll be using to build Clojure programs."
What's cool is that the functions (symbols) are automatically mapped into the user
namespace when the repl loads. That allows you to refer to the take-while symbol without some decorative
info before it.

In the Elixir case all of the [Elixir core](https://hexdocs.pm/elixir) is available but you reference
the functions using the module (i.e. `Enum.take_while`). In addition, there is an IExHelpers module
that is loaded that can be helpful in your repl sessions.

### String is a List?

Notice how in the Clojure code there is no step to convert the String into something that can be
processed by take-while. It just works. This is because in Clojure a String implements the ISeq
interface. So its not that the String is magically converted to a List. Its just that the String
is seq'able.

In Elixir, however, the String has to be converted to something that implements the Enumerable
protocol. A String does not. So we call `String.graphemes`.

### Doc?

In the Clojure repl I can get doc for a function by using `(doc func-name)`. So:

```
user=> (doc take-while)
-------------------------
clojure.core/take-while
([pred] [pred coll])
  Returns a lazy sequence of successive items from coll while
  (pred item) returns logical true. pred must be free of side-effects.
  Returns a transducer when no collection is provided.
nil
```

In the Elixir repl I can ask for help by using `h Mod.func`. So:

```
iex> h Enum.take_while

                        def take_while(enumerable, fun)

  @spec take_while(t(), (element() -> as_boolean(term()))) :: list()

Takes the items from the beginning of the enumerable while fun returns a truthy
value.

## Examples

    iex> Enum.take_while([1, 2, 3], fn x -> x < 3 end)
    [1, 2]
```

In addition in the Elixir repl I can do Enum. and hit tab and get the list of all the
functions available on the module. I haven't found the same thing in the Clojure repl
(but that certainly doesn't mean it doesn't exist since I'm a newbie). I prefer the
Elixir doc and help at this point but I do think that I need to become much more
familiar with Clojure doc and navigation in it's repl.

Ok, so that's a bunch of stuff based on a real easy problem. I have to say that I'm
enjoying exploring Clojure. I intend to spend some more time on it in the weeks to
come.