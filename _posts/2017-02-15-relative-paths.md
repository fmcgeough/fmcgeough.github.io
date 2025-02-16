---
layout: post
title: Elixir and Ruby expand_path
date: 2016-09-05 08:53:13
description: Features of Elixir
categories: Elixir
tags:
---

A common pattern in Ruby gem code is to have some sort of configuration file
(ordinarily in YAML format) stored in a config directory of the gem. The
gem then loads this with something along the lines of :

```
# return the contents of the system settings file as hash
def self.settings
  settings_file = File.expand_path(SYSTEM_CONFIG, __FILE__)
  YAML.load(File.read(settings_file))
end
```

Elixir has some of the same type of capabilities for expanding out a path
but you really wouldn't use them in the same way as you would in Ruby. If
you have built Phoenix projects you know about the config directory. One
thing you should have noticed is that all of those config files are actually
Elixir code. This is so much nicer.

But I did need to use the Elixir version of Ruby's expand_path for a seeds.exs
file. I wanted to reference a CSV file that the seeds.exs needed to read and
that was stored in the same directory. Since this was only used while developing
it worked fine. To do this I did :

```
    current_dir = Path.dirname(__ENV__.file)
    csv_file = "#{Path.expand("zip_codes.csv", current_dir)}"
```

So, a very similiar technique where the Path.expand method joins the directory
to the filename to give a full path.
