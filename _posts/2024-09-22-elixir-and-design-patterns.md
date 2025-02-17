---
layout: post
title: Elixir And Design Patterns
date: 2024-09-22 11:00:00
description: Elixir and using Design Patterns
tags:
categories: elixir
giscus_comments: true
---

Have you ever wondered about the answers you've given when someone asks you "what do you do?". If you are a software developer you may respond "I write software". Or if you know your company's goals you might say "I write software that ...insert one of the cooler sounding goals that you are aware of. Its when you are talking to someone else who spends their days editing, compiling, testing, debugging, releasing that you get into actual details. What is it that I do all day?

I'm interested in how companies create new software to solve a non-trivial problem. From my perspective there are several factors that contribute to a successful effort to solve it. One question I want to explore and hopefully answer is "is this a new problem or one that has already been solved?". If its a problem that has already been solved then I want to look at any previous solutions and answer the question: "is there a solution that is well understood as superior to others?". If there is one that is regarded as superior then I will use it (I'll also want to know why).

Is there a name for this type of thing? That is, a problem and a well understood general solution to the problem? Yes, its called a design pattern. Practically, a design pattern is "when I see problem xyz, I should use code that resembles abc".

Some sources for design patterns are:

- a design pattern that your organization has created. This tend to be large structural patterns. For example, your organziation may say "if any data needs to be stored it must be stored in a Postgres database" or "if any API is created it must adhere to the best REST API practices and include...".
- a design pattern that you have used before successfully to solve other similar problems. After you solve even your first problem you start building an internal mental catalog (if not a library) of what works and what doesn't.
- a design pattern created or influenced by the particular language, framework or tool you are using. If you are using Elixir and Phoenix (or other functional languages) then how you construct a solution is going to be different from a solution written in an object-oriented language.
- a design pattern book or paper that you think fits the problem. You can use a tool like ChatGPT (or similar) to describe the problem that you are trying to solve and ask it to design patterns that may be appropriate.

There has been a lot written about software design patterns. Software design patterns have been the subject of active discussions and numerous books since around 1977. There are two design patterns books that you will see referenced a lot. The first is "The Timeless Way of Building" by Christopher Alexander. Although this is a book about architecture it's had a big impact on creative thinking in a number of fields. The second book is software-centric. It's called "Design Patterns: Elements of Reusable Object-Oriented Software" by Erich Gamma, Richard Helm,Ralph Johnson, and John Vlissides (published by Addison-Wesley). These four authors were referred to as the "Gang of Four" (GoF). The book was released in 1994. (there's all sorts of other books, articles, papers of course but these two are frequently cited).

{% include figure.liquid loading="eager"
path="[assets/](https://github.com/fmcgeough/blog_posts/blob/main/)img/2024-09-design-patterns.jpg?raw=true"
class="img-fluid rounded z-depth-1" %}

A software design pattern is a description of a well-defined pattern. It is not an implementation that you can just copy/paste into your code. There are plenty of links to implementations for various languages that you can find with a simple search once you find a pattern that looks like it fits the problem you are trying to solve.

"Singleton" is an example of a design pattern. In the "Design Patterns" book this is described as "Ensure a class only has one instance, and provide a global point of access to it".

The description of the pattern may include a sample implementation. The "Design Patterns" book was written from a C++ perspective (Java was not released by Sun Microsystems to the world until 1995 - the year after the GoF book was published). So examples in that book are using C++. It's much more likely that you'll see Java example code nowadays.

In "Design Patterns" a design pattern has a number of common elements. I've listed them below. As mentioned the GoF book was written from the perspective of a C++ developer. This means that a number of the common elements are specific to object-oriented software. However, if you review this list you can perhaps see that a lot of the concepts can be applied to functional programs as well. The book is well worth reviewing even if you program entirely in a functional language.

- clear name - a description name that allows developers to use the term when discussing how it should or could be used in the software they are working on.
- context - from the GoF book the defined contexts are: creational, structural and behavioral.
- intent - what problem is this pattern solving? what are its goals?
- motivation - sometimes referred to as forces. Using a design pattern arises out of addressing common problems encountered when building systems. This explains why and when the pattern is ordinarily applied.
- applicability - a continuation of motivation but with an emphasis on the situations where the pattern is going to be the most effective.
- structure - the guts of the pattern. This may include class or interactive diagrams.
  - participants - key classes and objects in the pattern and what role they have in the pattern.
  - collaborations - how the classes and objects interact
- consequences - a discussion of the benefits and drawbacks to using the pattern.
- implementation - a concrete implementation may be provided in a particular language.
- known uses - description of any "real life" uses of the pattern that already exist
- related patterns - list of other patterns that this pattern may rely on or that may rely on this pattern.

Pattern discussions led to the development of the [Portland Pattern Repository](https://c2.com/ppr/titles.html). This used the WikiWikiWeb (this was the first user editable website - wiki - created by Ward Cunningham in 1995) to gather documentation on understood patterns. The Hillside Group also gathered patterns
together in their own [pattern catalog](https://hillside.net/patterns/patterns-catalog). There are numerous other pattern resources around the Internet and a number of other books that I'd recommend:

- "Refactoring: Improving the Design of Existing Code" by Martin Fowler
- "Domain-Driven Design: Tackling Complexity in the Heart of Software" by Eric Evans
- "Head First Design Patterns" by Eric Freeman and Bert Bates
- "The Art of Software Security Assessment" by Mark Dowd, John McDonald, and Justin Schuh

Once the idea of pattterns became widely accepted the concept of an anti-pattern arose. An anti-pattern in software engineering, project management, and business processes is a common response to a recurring problem that is usually ineffective and risks being highly counterproductive. In software the first use of the term seemed to be in 1995 by computer programmer Andrew Koenig. It seemed to get its first big public boost with the publication of the book Anti Patterns by The "Upstart Gang of Four": William Brown, Raphael Malveau, Skip McCormick, and Tom Mowbray.

{% include figure.liquid loading="eager"
path="[assets/](https://github.com/fmcgeough/blog_posts/blob/main/img/2024-09-antipatterns_book.jpeg?raw=true"
class="img-fluid rounded z-depth-1" %}

All of this is great and there is a lot of material to read and videos to watch on YouTube that can make you a better engineer and help you get better at problem solving and create better solutions. However, the bulk of it (almost all) is written from an object-oriented perspective (first C+++ and then Java). Translations of the patterns to other object-oriented languages (like Ruby) are pretty straightforward. That's not the case for a functional language like Elixir. I think functional programming does eliminate or greatly simplify a number of design patterns. So, are design patterns necessary for functional programming languages?

For me the answer is yes. If nothing else it helps to talk about the solution you are working on in a general way. For example, you may have state that you want to store in your application that should be stored as a "Singleton" Design pattern. The way you'd approach creating a singleton would be quite different from the same thing created in Java but the concept is the same. Someone with a background in design patterns can understand what it is you are describing and why you chose to use this pattern. There are also patterns that you are sure to use. For example, if you have an application that calls an API you may need a "Circuit Breaker" pattern. To protect your API you may want to utilize a "Throttle" pattern.

What resources on design patterns are available for an Elixir developer? It's a bit frustrating for people new to Elixir or functional programming in general. If you do a search in Github for "Design Patterns Java" you get 17.6k results. Do that same search for Elixir and you get 11.

Recently, more work is in process to explain the role of design patterns in Elixir. For example, a section on Antipatterns was added to the standard Elixir documentation. [Anti Patterns](https://hexdocs.pm/elixir/main/code-anti-patterns.html). There is a book called [Elixir Patterns](https://elixirpatterns.dev/) that is in development and will be released soon. José Valim gave a talk at ElixirConf EU 2024 on design patterns. Hopefully all of this work continues and accelerates. Here are some other links that may be useful.

- [Design Modeling Made Functional](https://pragprog.com/titles/swdddf/domain-modeling-made-functional/)
- [Gang of None](https://www.youtube.com/watch?v=agkXUp0hCW8)
- [Functional Programming Design Patterns](https://fsharpforfunandprofit.com/fppatterns/)
- [Typed Design Patterns for the Functional Era](https://arxiv.org/pdf/2307.07069)
