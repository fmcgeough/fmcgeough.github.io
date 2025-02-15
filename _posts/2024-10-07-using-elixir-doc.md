---
layout: post
title: Elixir and Documentation
date: 2024-10-16 10:00:00
description: Exploring functionality available in Elixir Documentation
tags:
categories: elixir
giscus_comments: true
---

I started learning about Elixir in 2016. One of the first things that I liked about it was the good-looking documentation. Examining important modules like [Process](https://hexdocs.pm/elixir/Process.html), [GenServer](https://hexdocs.pm/elixir/GenServer.html), or [Enum](https://hexdocs.pm/elixir/Enum.html) was a pleasure. There was a nice description of what the module provided and clear doc for each public function. The doc for functions would include example or explanatory code that was well formatted and easy to read.

I thought I'd try and summarize what I liked about the Elixir doc system when I first encountered it. The images shown below are from recent doc but you can "time-travel" back to previous versions of Elixir doc by selecting a version from the navigation. I believe this was added around 2019 (Elixir version 1.8).

## Elixir's Doc vs Java's Doc

I think what I liked about what I saw (over Java and Javadoc that I was working with in 2016) was it appeared it was written to be read. A big part of that was the format and flow. For example, here's the doc for Collections in Java JDK vs Enum in Elixir.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" 
        path="https://github.com/fmcgeough/blog_posts/blob/main/img/2024-10-java-collections.png" 
        class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Java Collections
</div>

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" 
        path="assets/img/2024-10-elixir-enum.png" 
        class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Elixir Collections
</div>

On the face of it these are presenting similar information. A name, a description, followed by different types of details. There's navigation for both.

The Java doc navigation shows the list of all packages in one frame and a list of all the available classes in the current package in another. The Elixir doc, on the other hand, has a tabbed navigation. One is "Modules" providing all the modules that are available in the library and other is "Pages".

The "Pages" tab has all sorts of goodies. It's got:

- API Reference - has a list of linked modules with a single sentence description
- Changelog (for the version you are looking at)
- Getting Started - probably the biggest section in "Pages" with coverage of topics that don't fit into doc for an individual module. These are general guides to language usage. For example, there's a section on "Basic Types" and "Anonymous Functions".
- Cheatsheets
- Anti-Patterns - this is relatively new list of things not to do
- Meta-Programming - one of Elixir's strength's is support for meta-programming. This allows developers to create DSL (Domain Specific Languages) that can simplify and clarify code
- Mix & OTP - Mix is Elixir's general purpose (and extensible) build tool. OTP is the system provided by the VM with core functionality and patterns that powers both Erlang and Elixir
- References - conventions, guidelines and more

The Elixir "Modules" tab was highlighted when I got to Enum. Enum appears in the modules documentation with three subheadings: Summary, Types, Functions. This is providing navigation that isn't available in Javadoc. You can expand the functions and click on any of them and the right-hand panel goes to the function and it's doc.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" 
        path="assets/img/2024-10-elixir-enum-nav-expanded.png" 
        class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Elixir Module Navigation
</div>

The code parts of the doc in Elixir (@specs, Examples, etc) show with a grey background and in a non-serif font. The Javadoc just has the font change. I find the Elixir version easier to read.

In both a function has a description. In Elixir this is broken into two pieces: a summary that appears first, followed by a break and then the actual description. In Javadoc the description is however many paragraphs of text are needed to describe the function. For example, for both a Java Collection and Enum there is a min function. For Elixir the description is "Returns the minimal element in the enumerable according to Erlang's term ordering". For Javadoc its "Returns the minimum element of the given collection, according to the natural ordering of its elements. All elements in the collection must implement the Comparable interface. Furthermore, all elements in the collection must be mutually comparable (that is, e1.compareTo(e2) must not throw a ClassCastException for any elements e1 and e2 in the collection)".

Both of these are providing useful information but it's easy to see that the Elixir doc is an easier on-ramp to learning what the code provides. The Javadoc provides lots of useful information however it definitely reads like it was written by a lawyer. My preference is for the Elixir style.

The Java doc appears crowded. There's links everywhere. By contrast the Elixir doc is quite clean.

## Elixir Doc vs Other Modern Languages

You might think that Elixir doc might be better but that's because it was invented rather recently. However, it's more than that. You can look at another couple of "recent" languages and see that their doc is not as clear and easy to use as Elixir. I'll use Rust and Go.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" 
        path="https://github.com/fmcgeough/blog_posts/blob/main/img/2024-10-rust-collections.png" 
        class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Rust Collections
</div>

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" 
        path="assets/img/2024-10-go-collections.png" 
        class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Go Collections
</div>

I'm not saying these languages are not useful (by any stretch). They are both amazing languages. But their target audience seems to be quite different. And, in both cases, I think it's fair to say that the doc is provided as a reference. That is, if you already know how everything works but need some piece of individual information then it's useful. Trying to learn by looking at this doc is the wrong approach. There are quite good books and blog posts that can help in that regard.

The ability to go to source code from documentation was something I was familiar with in Go before I ever looked at Elixir. I was happy that the Elixir creators added this capability. I find it quite useful.

## iex and Documentation

Elixir comes with a repl like Python or Ruby. The repl is called iex. When you are developing locally and using [iex](https://hexdocs.pm/iex/IEx.html) you can access documentation. It does require that you "know" what you are looking for but provides a bit of help in that regard.

As an example let's look at DateTime. If you are using DateTime in the iex repl and forget what functions are available you can enter `DateTime.` and hit `<tab>`. All the functions are displayed.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" 
        path="assets/img/2024-10-iex_module_functions.png" 
        class="img-fluid rounded z-depth-1" %}
    </div>
</div>

Since DateTime has a large number of functions, all of the possible functions are not displayed. You can use `<pg-up>` or `<pg-down>` to show all the functions.

To get help on any individual function you can use `h` followed by the function name (and possibly arity if there are multiple functions with same name but different arity).

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" 
        path="assets/img/2024-10-iex-function-help.png" 
        class="img-fluid rounded z-depth-1" %}
    </div>
</div>

This extensive help is available in iex for not only the core Elixir software but any important library that you have a dependency on (Phoenix, Ecto, etc) when you are working on your own project.

One last thing is that there is tab completion for finding a module as well. So if I type "D" and `tab` then I get Date, DateTime, Dict, Duration, and DynamicSupervisor. This is not only useful for lookup but things like this that save having to type are always welcome if you develop software for a living.

## Core Elixir Doc

There is a Documentation page on the main Elixir site at
[docs.html](https://elixir-lang.org/docs.html). Elixir is broken into 6 different applications:

- Elixir - standard library
- EEx - templating library
- ExUnit - unit test library
- IEx - interactive shell
- Logger - built-in Logger
- Mix - build tool

The page displays links for multiple Elixir versions and indicates what the supported Erlang/OTP versions are for each Elixir version.

You can visit the [DateTime doc](https://hexdocs.pm/elixir/DateTime.html) and see that the [convert/2 function](https://hexdocs.pm/elixir/DateTime.html#convert/2) covered above. Notice that the doc that appears on that page is the same doc that shows up in iex when you ask for help on the function.

## Library doc and hex.pm

Libraries for both Erlang and Elixir are available via [hex.pm](https://hex.pm/). The same tool that produced the Elixir core library documentation is used by library authors (ex_doc). The library used to generate doc - ex_doc - is also going to be in hex.pm.

Let's examine a fundamental Elixir library - [plug](https://hex.pm/packages/plug). This is the basis for how the Phoenix web framework works. If you search for plug in hex.pm you can navigate to its page. You'll see a lot of information on this main page.

- Links
  - Online documentation (library documentation). Next to this link is a little image that lets you download all the library doc to your local system as a .tar.gz file.
  - GitHub (where code is actually stored)
- Downloads
  - Displays a graph of how many times the library has been downloaded. It also shows general info for number of downloads in certain time frames (yesterday, last week, all time).
- Versions
  - Each published version is displayed here with the version number, date published and links to the documentation for that particular version.
- Dependencies
  - A library may use other libraries. If so this section lists off what libraries this library is dependent on and what the version is of the required library. Optional dependencies such as libraries used for testing the library not listed. This list is meant to give you an overview of what libraries your deployed code will have if you use the library.
- Recent Activity
  - This shows important recent events for the library.
- Config
  - This section shows you how to install the library for your project. It includes what you'd add to your mix.exs file (or rebar.config/erlang.mk if you are using Erlang)
- Checksum
  - This has the checksum for the library that was published
- Build Tools
  - This lists what is used to build the library. This will be `mix` for Elixir libraries or `rebar3` for Erlang libraries (ordinarily).
- Owners
  - This shows the list of developers that are the "owners" of the library.
- Publisher
  - This shows the individual who is allowed to publish a new version of the library
- Dependants
  - This shows libraries that are dependent on this library. In the case of the `plug` library this is a very long list so the text ends with "..." indicating that there are more available than are listed. You can click on the last library listed and get a paged list of dependencies.

The [plug library documentation page](https://hexdocs.pm/plug/readme.html) looks like this:

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" 
        path="https://github.com/fmcgeough/blog_posts/blob/main/img/2024-10-plug_documentation.png" 
        class="img-fluid rounded z-depth-1" %}
    </div>
</div>

There are a few general characteristics of the generated doc that are important to note:

- the left hand navigation consists of two tabs: 1) Pages; 2) Modules. The Pages contains various text that supports the library and describes use cases. The Modules tab lists the modules that are documented in the library.
- if you want to resize the left-hand navigation there is a small widget at the bottom right of the navigation that allows dragging the left hand navigation to widen or narrow it.
- The search functionality is full-text JS based. Here are some tips:
  - Multiple words (such as foo bar) are searched as OR
  - Use _ anywhere (such as fo_) as wildcard
  - Use + before a word (such as +foo) to make its presence required
  - Use - before a word (such as -foo) to make its absence required
  - Use : to search on a particular field (such as field:word). The available fields are title, doc and type
  - Use WORD^NUMBER (such as foo^2) to boost the given word
  - Use WORD~NUMBER (such as foo~2) to do a search with edit distance on word
- next to the search entry there is a widget that allows you to change the theme (along with a couple of other settings).
- to the right of module or functions there is a widget `<>` that can be clicked on. It brings you to the location in the source code (in GitHub, ordinarily) where the doc occurs. This is quite handy for navigating to source code if you are curious about how something is implemented.

## Wrap Up

I was genuinely impressed when I started looking at Elixir back in 2016. It's clear that there was a set of goals related to documentation when the language was developed. I found the doc quite useful compared to other languages that I was looking at or working with at the time. The images shown are from documentation now (not 2016). Many of the same things were already in place in Elixir back then.

I liked the layout (which has improved quite a bit from 2016). I appreciated the organization of the material. If I found any issues in the documentation I was able to get a pull request merged rather quickly (usually in a couple hours).

These are all impressions before I tried using the documentation system myself. I'll write another post covering writing documentation in Elixir.
