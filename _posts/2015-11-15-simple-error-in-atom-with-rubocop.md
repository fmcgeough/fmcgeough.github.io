---
layout: post
title: Simple Error in Atom with Rubocop
date: 2015-11-15 08:53:13
description: Information on running rubocop in the Atom editor
tags: ruby
categories: ruby
---

If you keep getting popups errors in the github [Atom text editor](https://atom.io)
when using [Rubocop](https://atom.io/packages/linter-rubocop) then maybe its this issue I just had. While working on a gem complaining about a missing rubocop version or in general giving odd and annoying errors you should try and remove your Gemfile.lock file in the root of your gem folder and run [bundle](http://bundler.io/v1.3/rationale.html) install again. It could be that the Gemfile.lock is pointing at a rubocop you no longer have installed. I was frustrated for several minutes about these errors when I brought up an older gem to test out Rubocop integration with Atom until I realized that I probably had done a bundle install way back when I wrote this gem and created the Gemfile.lock file which the Rubocop was trying to respect.
