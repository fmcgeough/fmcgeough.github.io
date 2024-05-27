---
layout: post
title: Using Git From Inside Atom
date: 2015-11-22 08:53:13
description: Information on using git in the Atom editor
tags: ruby
categories: ruby
---

I continue to use and be impressed with the [Atom editor](https://atom.io/) from the lovely people at [github](https://github.com/). A modern text editor with a built-in packaging system to allow adding in, well, add-ins that provide cool and needed functionality. One of the new ones that I'm using a lot is [git-plus](https://atom.io/packages/git-plus).

git-plus allows me to work within the text editor and perform git commands without dropping to the command shell. This is a small thing for me. I'm quite comfortable with using git from the command line but it actually does make my flow better while coding in Ruby.

My Environment and Install

- OS X 10.10.5
- Atom 1.2.2
- git-plus 5.6.5

I installed git-plus from the command line using the Atom package manager utility apm.

```
    apm install git-plus
```

Then I restarted Atom from the command line to use the new package. There were no issues with the package install or initial usage.

There are a number of settings in git-plus to allow customization. You can find them under Preferences/Packages in Atom. I didn't modify any of the defaults but as noted on the Package Settings page you should : "Make sure your gitconfig file is configurated or at least your user.email and user.name variables are initialized". The web page for the git-plus package has a good description of the available configuration options.

To bring up the git-plus functionality you use :

```
    Cmd-Shift-H on MacOS
```

This brings up a searchable list of git commands (with "Add" at the top). As you type the list narrows. Hit Enter to select the git command. On a "Commit" a text panel appears on the right side of the Atom environment. Add your commit message and save it. The panel disappears and a notification pops up to let you know your changes are committed. The notification auto-closes after a couple seconds.

## Why Use Git from Within a Tool?

I sort of struggle with this myself. I work with a lot of Java engineers who tend to live within their IDE (Eclipse or IntelliJ) and are actually quite at sea when trying to use git from the command line. That's not a good thing. If you are using git you absolutely need to be familiar with common use cases from the command line. git was developed as a command line program. You need to live in its environment or ultimately bad things will occur and you'll have no clue how to address them.

If I'm programming in Java I appreciate the help that a good IDE can provide for the language - especially in terms of intelligent search and refactoring. However, for Ruby I've never really embraced an IDE and have continued to rely on a text editor, and I've grown to really like the Atom editor for Ruby.

The reason I've grown to also like the git-plus package in Atom is that it doesn't tend to interrupt my flow when I'm working on Ruby code and accompanying unit tests. I can quickly check in work with less of an interruption to my thought processes. I didn't think this would be so and I was ready to uninstall git-plus before I even installed it for that reason but I was surprised to find that it actually did seem to make me more productive or at least it felt that way.

Its another well-done package for the Atom editor and I'd definitely recommend trying it to see if it works well for you.
