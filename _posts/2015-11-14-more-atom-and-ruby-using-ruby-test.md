---
layout: post
title: More Atom and Ruby - Using ruby-test
date: 2015-01-14 08:53:13
description: Using Ruby with the Atom editor
tags: ruby
categories: ruby
---

I continue to try and get [Atom](https://atom.io/) setup in the most productive
way - especially for Ruby programming. I continue to be impressed with the tool.
For unit testing I favor [MiniTest](https://github.com/seattlerb/minitest) and I
wanted a way to run tests from inside Atom without leaving the editor environment.

After trying out of a couple of packages I settled on [ruby-test](https://atom.io/packages/ruby-test).

## Installation

Installation of ruby-test is straightforward. I used the command line Atom
install tool `apm`. The following worked for me :

apm install ruby-test

After installing the ruby-test package restart Atom by using `atom` from the
command line and then configure the testing package by going to
Atom/Preferences/Packages and then searching for ruby-test. You'll want to set
your "Test Framework" as explained in the configuration panel.

The tool worked seamlessly for me after doing this.

## Review

Install was straightforward and the tool within [Atom](https://atom.io/) worked
seamlessly for me. My environment is :

- OS X 10.10.5 (Yosemite)
- Atom 1.2.0
- ruby-test 0.9.16

The tool pops up a panel when running tests and shows the MiniTest output
just like you'd see from the command line. The panel has a close button
(unlike some of the other unit test packages I tried out in Atom) so you can
close the unit test info when you no longer need it or want additional screen
real estate. The panel that pops up includes a link in the left hand corner to
"Settings" that brings up you to the ruby-test Settings page.

I generally run a single test after I finish writing it by placing my cursor on
the test and hitting command-control-R and then run all the tests in a Ruby test
file by hitting command-control-T. Both of these approaches worked without issue.
