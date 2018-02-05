---
layout: post
title: 'Elixir/Phoenix - shared templates'
categories:
- Elixir
tags: []
---
In using server-side rendering for Phoenix web apps there are two approaches that
I've used for dealing with removing duplication from templates that need to render
the same information. The first is within a single view and the other is across views.

The thing that is important to understand about templates is that the html.eex that
you are writing is compiled and the show or index or whatever your template name is becomes
a function in your view. So, you have your view modules in your views directory (by default)
and the templates directory is organized by the view name.

This is covered in the [Phoenix documentation](https://hexdocs.pm/phoenix/templates.html).

If you want to split up a template in one of your template directories
into multiple files you can do so. In place of the code you remove from your
template that you are splitting you'll need a render statement. Let's
say that I have a tabbed display for my show page where one tab shows general
descriptive information and another displays a table of tags associated with the
item that the user is viewing. I could do take the template code that generates the HTML
and throw into two files in a subdirectory under my main template directory:

```
elements/description.html.eex
elements/tags.html.eex
```

Then in my show.html.eex I can do :

```
<div class="tab-content clearfix">
  <div class="tab-pane active" id="description">
    <%= render "elements/description.html", assigns %>
  </div>
  <div class="tab-pane" id="tags">
    <%= render "elements/tags.html", assigns %>
  </div>
</div>
```

Lets say you wanted to share template code across different views. For example, suppose there
were navigation buttons that you wanted to display on a subset of pages. The buttons will always
be the same. You shouldn't duplicate that template code in each of the template directories.
Rather, you should create a new SharedView with a corresponding shared template directory and
put your shared template code there. Rendering that in each of the separate template directories
then becomes:

```
<%= render Elixir.MyAppWeb.SharedView, "navigation_buttons.html", assigns) %>
```
