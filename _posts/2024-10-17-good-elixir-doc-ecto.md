---
layout: post
title: Good Elixir Documentation - Ecto Library
date: 2024-10-18 09:00:00
description: How the Ecto Library Uses ex_doc to generate doc
tags:
categories: elixir
giscus_comments: true
---

This is a continuation of my blog post on [Elixir and Documentation](https://fmcgeough.github.io/blog/2024/using-elixir-doc/). I wanted to go over some of the projects that I think have very good documentation. When you are writing your own doc it's helpful to have good examples to work from. The one that I'm writing about first is Ecto. _Note: the Elixir language documentation is also a great resource_.

The [ecto library](https://hexdocs.pm/ecto/Ecto.html) is the relational database library used in Elixir. The doc it generates has some interesting features. There are 3 tabs in the Navigation:

- Guides
- Modules
- Mix Tasks

## Guides

The guides section contains an Introduction section that explains the basics of how to use Ecto and how to do unit testing with Ecto. It has Cheatsheets for basic Ecto related operations and for how to handle table associations. A final section of How-To's covers a range of topics in more detail.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" 
        path="assets/img/2024-10-17-ecto-guides.png" 
        class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Ecto Guides
</div>

## Modules

The modules section has the documentation for modules and their functions. It also covers Types and Exceptions. The modules are actually broken into logical areas of functionality:

- Query APIS
- Adapter Specification
- Relation Structs
- Exceptions

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" 
        path="assets/img/2024-10-17-ecto-modules.png" 
        class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Ecto Modules
</div>

## Mix Tasks

There are important mix tasks associated with Ecto that allow a developer to create, migrate or drop a database. There is also a task to generate code related to an Ecto Repo module.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" 
        path="assets/img/2024-10-17-ecto-mix-tasks.png" 
        class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Ecto Mix Tasks
</div>

## Ecto mix.exs

The Ecto mix.exs file has a lot of lines related to documentation. Since it's a lot of lines I'll break it into sections.

## the project function

The ex_doc library reads the project data `:name` and `:docs` values. Using a function to define the data for `:docs` is how projects generally define documentation data in mix.exs. It's overwhelming if all the `:docs` lines are part of the project function.

```
  def project do
    [
      app: :ecto,
      version: @version,
      elixir: "~> 1.11",
      deps: deps(),
      consolidate_protocols: Mix.env() != :test,
      elixirc_paths: elixirc_paths(Mix.env()),

      # Hex
      description: "A toolkit for data mapping and language integrated query for Elixir",
      package: package(),

      # Docs
      name: "Ecto",
      docs: docs()
    ]
  end
```

## docs elements

The elements that can be used in `:docs` are documented in the [ex_doc Mix Tasks](https://hexdocs.pm/ex_doc/Mix.Tasks.Docs.html). An abbreviated version is shown below:

- :annotations_for_docs - a function that receives metadata and returns a list of annotations to be added to the signature.
- :api_reference - Whether to generate api-reference.html; default: true. If this is set to false, :main must also be set.
- :assets - A map of source => target directories that will be copied as is to the output path. It defaults to an empty map.
- :authors - List of authors for the generated docs or epub.
- :before_closing_body_tag - a function that takes as argument an atom specifying the formatter being used (:html or :epub) and returns a literal HTML string to be included just before the closing body tag (</body>).
- :before_closing_head_tag - a function that takes as argument an atom specifying the formatter being used (:html or :epub) and returns a literal HTML string to be included just before the closing head tag (</head>). The atom given as argument can be used to include different content in both formats. Useful to inject custom assets, such as CSS stylesheets.
- :before_closing_footer_tag - a function that takes as argument an atom specifying the formatter being used (:html) and returns a literal HTML string to be included just before the closing footer tag (</footer>).
- :canonical - String that defines the preferred URL with the rel="canonical" element; defaults to no canonical path.
- :cover - Path to the epub cover image (only PNG or JPEG accepted) The image size should be around 1600x2400.
- :deps - A keyword list application names and their documentation URL. ExDoc will by default include all dependencies and assume they are hosted on HexDocs. This can be overridden by your own values. Example: `[plug: "https://myserver/plug/"]`
- :extra_section - String that defines the section title of the additional Markdown and plain text pages; default: "PAGES". Example: "GUIDES"
- :extras - List of paths to additional Markdown (.md extension), Live Markdown (.livemd extension), Cheatsheets (.cheatmd extension) and plain text pages to add to the documentation.
- :filter_modules - Include only modules that match the given value. The value can be a regex, a string (representing a regex), or a two-arity function that receives the module and its metadata and returns true if the module must be included. If a string or a regex is given, it will be matched against the complete module name (which includes the "Elixir." prefix for Elixir modules). If a module has @moduledoc false, then it is always excluded.
- :formatters - Formatter to use; default: ["html", "epub"], options: "html", "epub".
- :groups_for_extras, :groups_for_modules, :groups_for_docs - See the "Groups" section
- :ignore_apps - Apps to be ignored when generating documentation in an umbrella project. Receives a list of atoms. Example: [:first_app, :second_app].
- :language - Identify the primary language of the documents, its value must be a valid BCP 47 language tag; default: "en"
- :logo - Path to a logo image file for the project. Must be PNG, JPEG or SVG.
- :main - Main page of the documentation. It may be a module or a generated page, like "Plug" or "api-reference"; default: "api-reference".
- :markdown_processor - The markdown processor to use, either module() or {module(), keyword()} to provide configuration options;
- :meta - A keyword list or a map to specify meta tag attributes
- :nest_modules_by_prefix - See the "Nesting" section
- :output - Output directory for the generated docs; default: "doc". May be overridden by command line argument.
- :skip_undefined_reference_warnings_on - ExDoc warns when it can't create a Mod.fun/arity reference in the current project docs e.g. because of a typo. This list controls where to skip the warnings, for a given module/function/callback/type (e.g.: ["Foo", "Bar.baz/0"]) or on a given file (e.g.: ["pages/deprecations.md"]).
- :skip_code_autolink_to - Similar to :skip_undefined_reference_warnings_on, this option controls which terms will be skipped by ExDoc when building documentation.
- :source_beam - Path to the beam directory; default: mix's compile path.
- :source_ref - The branch/commit/tag used for source link inference; default: "main".
- :source_url_pattern - Public URL of the project for source links.

## the docs function

Ecto defines an extensive `:docs` element. It uses almost every available option in ex_doc. One thing it does not override is `:output` (by default this is the `./doc` directory). Notice that the version and source_url are set using module attributes (`@version` and `@source_url`). This is a good practice (especially for a library) since those values are useful elsewhere in mix.exs. The `@version` attribute is used in the project and the `@source_url` is used in the package function.

```
  defp docs do
    [
      main: "Ecto",
      source_ref: "v#{@version}",
      logo: "guides/images/e.png",
      extra_section: "GUIDES",
      source_url: @source_url,
      skip_undefined_reference_warnings_on: ["CHANGELOG.md"],
      extras: extras(),
      groups_for_extras: groups_for_extras(),
      groups_for_docs: [
        group_for_function("Query API"),
        group_for_function("Schema API"),
        group_for_function("Transaction API"),
        group_for_function("Process API"),
        group_for_function("Config API"),
        group_for_function("User callbacks")
      ],
      groups_for_modules: [
        Types: [
          Ecto.Enum,
          Ecto.ParameterizedType,
          Ecto.Type,
          Ecto.UUID
        ],
        "Query APIs": [
          Ecto.Query.API,
          Ecto.Query.WindowAPI,
          Ecto.Queryable,
          Ecto.SubQuery
        ],
        "Adapter specification": [
          Ecto.Adapter,
          Ecto.Adapter.Queryable,
          Ecto.Adapter.Schema,
          Ecto.Adapter.Storage,
          Ecto.Adapter.Transaction
        ],
        "Relation structs": [
          Ecto.Association.BelongsTo,
          Ecto.Association.Has,
          Ecto.Association.HasThrough,
          Ecto.Association.ManyToMany,
          Ecto.Association.NotLoaded,
          Ecto.Embedded
        ]
      ],
      before_closing_body_tag: fn
        :html ->
          """
          <script src="https://cdn.jsdelivr.net/npm/mermaid@10.2.3/dist/mermaid.min.js"></script>
          <script>
            document.addEventListener("DOMContentLoaded", function () {
              mermaid.initialize({
                startOnLoad: false,
                theme: document.body.className.includes("dark") ? "dark" : "default"
              });
              let id = 0;
              for (const codeEl of document.querySelectorAll("pre code.mermaid")) {
                const preEl = codeEl.parentElement;
                const graphDefinition = codeEl.textContent;
                const graphEl = document.createElement("div");
                const graphId = "mermaid-graph-" + id++;
                mermaid.render(graphId, graphDefinition).then(({svg, bindFunctions}) => {
                  graphEl.innerHTML = svg;
                  bindFunctions?.(graphEl);
                  preEl.insertAdjacentElement("afterend", graphEl);
                  preEl.remove();
                });
              }
            });
          </script>
          """

        _ ->
          ""
      end
    ]
  end
```

## skip_undefined_reference_warnings_on

The defined value for `:skip_undefined_reference_warnings_on` is set
to `["CHANGELOG.md"]`. This makes sense. There may be situations where a module or type is removed from the code base. This will be noted in the CHANGELOG but if it is and we don't set that file in the `:skip_undefined_reference_warnings_on` list then warnings are generated.

## using extras function

The ex_doc library describes how to use both `:extras` and `groups_for_extras`. These value are used to by Ecto to provide useful information under the "GUIDES" tab. Ecto uses this:

```
  extras: extras(),
  groups_for_extras: groups_for_extras(),
```

This defines both of those values for the two keys with whatever is returned by those functions.

## the extras

Any files that you want to include in your doc that are not in modules must be listed under `:extras`. For Ecto this is:

```
  def extras() do
    [
      "guides/introduction/Getting Started.md",
      "guides/introduction/Embedded Schemas.md",
      "guides/introduction/Testing with Ecto.md",
      "guides/howtos/Aggregates and subqueries.md",
      "guides/howtos/Composable transactions with Multi.md",
      "guides/howtos/Constraints and Upserts.md",
      "guides/howtos/Data mapping and validation.md",
      "guides/howtos/Dynamic queries.md",
      "guides/howtos/Multi tenancy with query prefixes.md",
      "guides/howtos/Multi tenancy with foreign keys.md",
      "guides/howtos/Self-referencing many to many.md",
      "guides/howtos/Polymorphic associations with many to many.md",
      "guides/howtos/Replicas and dynamic repositories.md",
      "guides/howtos/Schemaless queries.md",
      "guides/howtos/Test factories.md",
      "guides/cheatsheets/crud.cheatmd",
      "guides/cheatsheets/associations.cheatmd",
      "CHANGELOG.md"
    ]
  end
```

Notice how all the "extra" doc is under the "guides" directory. The sections are separate directories under "guides". That is, "introduction", "howtos" and "cheatsheets". The CHANGELOG.md is also listed here since Ecto wants that included in the generated documentation.

## Ecto mix.exs - groups_for_extras

In the generated Ecto doc under "GUIDES" there are sections:

- "INTRODUCTION"
- "CHEATSHEETS"
- "HOW-TO'S"

These are generated using `:groups_for_extras`. Ecto defines the value for this in the mix.exs as:

```
  defp groups_for_extras do
    [
      Introduction: ~r/guides\/introduction\/.?/,
      Cheatsheets: ~r/cheatsheets\/.?/,
      "How-To's": ~r/guides\/howtos\/.?/
    ]
  end
```

Here, Ecto is using a regex to match the files in the guides subdirectories. The key in the returned Keyword list becomes the section title.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" 
        path="assets/img/2024-10-17-ecto-guides.png" 
        class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Ecto Guides
</div>

## using Groups

The ex_doc library describes [how to use the various Groups functions](https://hexdocs.pm/ex_doc/Mix.Tasks.Docs.html#module-groups) in it's documentation.

## groups_for_docs

There is a defined value for `:groups_for_docs` that sets of six groups.

```
  group_for_function("Query API"),
  group_for_function("Schema API"),
  group_for_function("Transaction API"),
  group_for_function("Process API"),
  group_for_function("Config API"),
  group_for_function("User callbacks")
```

The `group_for_function/1` function is:

```
defp group_for_function(group), do: {String.to_atom(group), &(&1[:group] == group)}
```

This may look a bit odd if you haven't used it before. What it's doing is allowing any documented function to declare a doc group. If that is found then the function becomes part of that named group. In the case of Ecto the `:groups_for_docs` value is being used to help organize the functions under `Ecto.Repo` (which has a particularly wide API). For example, the `get/2` callback in Repo:

```
  @doc group: "Query API"
  @callback get(
    queryable :: Ecto.Queryable.t(),
    id :: term, opts :: Keyword.t()
  ) :: Ecto.Schema.t() | term | nil
```

If you look at the `Ecto.Repo` doc navigation you'll see:

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" 
        path="assets/img/2024-10-18-ecto-repo-groups.png" 
        class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Ecto.Repo Function Grouping
</div>

## groups_for_modules

There is a defined value for `:groups_for_modules`. This value allows you to logically group the modules in your project doc. This shows up in the "Modules" tab of the generated doc. For Ecto the groups are:

- Types
- Query APIs
- Adapter specification
- Relation structs

You can use a regex when specifying what modules belong to a group. However, for the Ecto doc the modules are provided as a list. If you can use a regex here you should. A regex is effective if your modules are stored in subdirectories that match your group naming.

If a module does not match any file in `:groups_for_modules` (and that module does not have a `@moduledoc false`) then the module shows up at the top of the "Modules". For Ecto this is `Ecto`, `Ecto.Changeset`, `Ecto.Multi`,
`Ecto.Query`, `Ecto.Repo`, `Ecto.Schema`, `Ecto.Schema.Metadata` and `Mix.Ecto` (as of version 3.12.4).

## before_closing_body_tag

There is a defined value for `:before_closing_body_tag`. This defines Javascript used to allow [mermaid.js](https://mermaid.js.org/) to work in the generated documentation. A mermaid generated diagram is available for the type [Ecto.Type](https://hexdocs.pm/ecto/Ecto.Type.html).

This code could be copied into your own project if you wish to use mermaid generated diagrams in your doc.

```
      before_closing_body_tag: fn
        :html ->
          """
          <script src="https://cdn.jsdelivr.net/npm/mermaid@10.2.3/dist/mermaid.min.js"></script>
          <script>
            document.addEventListener("DOMContentLoaded", function () {
              mermaid.initialize({
                startOnLoad: false,
                theme: document.body.className.includes("dark") ? "dark" : "default"
              });
              let id = 0;
              for (const codeEl of document.querySelectorAll("pre code.mermaid")) {
                const preEl = codeEl.parentElement;
                const graphDefinition = codeEl.textContent;
                const graphEl = document.createElement("div");
                const graphId = "mermaid-graph-" + id++;
                mermaid.render(graphId, graphDefinition).then(({svg, bindFunctions}) => {
                  graphEl.innerHTML = svg;
                  bindFunctions?.(graphEl);
                  preEl.insertAdjacentElement("afterend", graphEl);
                  preEl.remove();
                });
              }
            });
          </script>
          """

        _ ->
          ""
      end
```

## Why does CHANGELOG.md become Changelog for v3.x?

There is nothing in mix.exs indicating that ex_doc should change the name of CHANGELOG.md to the name that actually shows in the generated doc "Changelog for v3.x" so how is this accomplished? If you open CHANGELOG.md you can see where this is coming from.

```
# Changelog for v3.x

## v3.12.4 (2024-10-07)

### Enhancements

  * [Ecto.Repo] Use `persistent_term` for faster repository lookup
  * [Ecto.Repo] Document new `:pool_count` option
etc, etc
```

The top-level heading is being used by ex_doc as the name that shows up in the navigation.

## what about "MIX TASKS"?

There is a separate tab in navigation called "MIX TASKS". Where did that come from? It's not mentioned explicitly in the mix.exs file.

The Ecto mix tasks are under `lib/mix/tasks`. The ex_doc library recognizes this as a "special" thing and puts the doc that is in the modules in that directory into the "MIX TASKS" tab.

## Wrap Up

You can examine the [ex_doc mix.exs file](https://github.com/elixir-ecto/ecto/blob/master/mix.exs).
