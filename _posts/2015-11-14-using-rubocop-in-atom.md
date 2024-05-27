---
layout: post
title: Using Rubocop in Atom
date: 2015-11-14 08:53:13
description: Information on running rubocop in the Atom editor
tags: ruby
categories: ruby
---

If you haven't checked out the [Atom editor](https://atom.io) from the lovely
people at github then its worth your while. A full-featured and more and more
popular editor with a growing set of packages that can be added in to provide
some cool functionality. Since I'm doing more and more Ruby programming I
wanted to install the [rubocop](https://github.com/bbatsov/rubocop) lint
functionality within the Atom editor. This is provided by the [linter-rubocop](https://atom.io/packages/linter-rubocop) Atom package.
So this blog post might be interesting if you code in Ruby and are open to
trying a new text editor.

## What's Rubocop?

Rubocop is a static code analysis tool that tries to report on any cases where your code conflicts with the Ruby community style guide. I find it very valuable to try and keep my code clean and easier to maintain. It is a source of tremendous exasperation when you first use it (at least that has been my experience when getting people to use it) because it will spit out dozens (or hundreds) of problems with your code. I find working through these issues is very valuable though and definitely recommend running this tool on your own projects.

## Installation

Assuming you just went through the standard Atom installer from their web site you now have a command line utility called apm. This allows you to install Atom packages. To get the rubocop to work in Atom you'll need rubocop, linter (the base lint package that is used by the linter-rubocop package, and linter-rubocop. The following steps should get you working :

    gem install rubocop
    apm install linter
    apm install linter-rubocop
    which rubocop

The final command will tell you your path to rubocop. You can paste this into the configuration for linter-rubocop. Now that you have these installed you should start the Atom editor from the command line. Now as you type your Ruby code you'll see information in the bottom panel of your editor screen (that hopefully says "No Issues").

## Review

I didn't have any issues installing the package using the steps above (I did already have rubocop installed). I run the Atom editor on Mac OS X. The versions I used are :

- Atom 1.2.0
- rubocop 0.34.2
- linter-rubocop 0.4.4
- linter 1.11.1

I did see an error when starting Atom using Quicksilver :

There are a variety of posts about this on the web. It doesn't appear when starting Atom from the command line (which is how I ordinarily use the tool).
