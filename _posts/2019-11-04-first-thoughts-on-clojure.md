---
layout: post
title: 'First thoughts on learning Clojure'
categories:
- Clojure
tags: []
---
I decided to spend some time learning Clojure. I figured I'd install and find some good doc or
an online book to work with. The install itself was quite painless. The computer I'm using for
this is a Macbook pro and I've got homebrew installed. You'll need to install Java (which I already
had) and then installing Clojure is just `brew install clojure`. After reading the docs one of the
first things you'll realize is that you need a tool called [leiningen](https://leiningen.org/). Installing
that is also just `brew install leiningen`. The online book I found is [Clojure for the Brave and True](https://www.braveclojure.com). Its funny and well organized and seems up to date enough. I also signed up for an
[exercism account](https://exercism.io/my/tracks/clojure) which has a Clojure track for algorithm
practice once I get some actual chops with the language itself.

At the moment here's the real limited amount of things I know so far:

* There's a repl in Clojure
* The syntax is Lisp like.  opening parenthesis, operator, operands, closing parenthesis.
* The [Clojure web site](https://clojure.org/index) is pretty and looks well organized.
* There's a [Slack Channel](http://clojurians.net/)
* Data structures are not the focus (It is better to have 100 functions operate on one data structure than 10 functions on 10 data structures. —Alan Perlis). This is a common functional programming language mantra.
* Functions are first class citizens
* Error messages are computereese

```
clojure-noob.core=> ("test" 1 2 3)
Execution error (ClassCastException) at clojure-noob.core/eval1673 (form-init15352419622516719568.clj:1).
class java.lang.String cannot be cast to class clojure.lang.IFn (java.lang.String is in module java.base of loader 'bootstrap'; clojure.lang.IFn is in unnamed module of loader 'app')
```
* This is saying that "test" isn't a function.

Going to continue working thru the book (which again seems terrific!) and see how far I get into Clojure-land. Still
Clojure-curious I guess.
