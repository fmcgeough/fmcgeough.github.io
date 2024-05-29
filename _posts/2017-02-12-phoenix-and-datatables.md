---
layout: post
title: Elixir, Phoenix Framework and Datatables
date: 2017-02-12 08:53:13
description: Exploring Features of Elixir
categories: Elixir
tags: Elixir
---

I built an administrative tool for my company that used Phoenix server side templating,
bootstrap and datatables.js. This is not because I have an overwhelming preference for bootstrap
or datatables.js. I just happened to find these first in the course of building the web app and
they were able to solve my problem. Phoenix server-side templates are awesome, powerful and
easy-to-use. I did have a bit of trouble with datatables so I figured I'd write up a brief blog post on how
I got datatables to work with server-side paging.

The rundown on what is used for this project is :

- Elixir "1.3.3"
- Phoenix "1.2.1"
- Ecto "2.1.3"

All of the code is on github.com at [https://github.com/fmcgeough/datatables](https://github.com/fmcgeough/datatables)

Datatables has the ability to do server-side paging via AJAX calls. There is enough
information on the web to be able to puzzle out how to do this but I didn't see any Elixir/Phoenix examples
so I figured I'd do a simple one in case someone else is looking for this information. Its
comforting to find an implementation in your language of choice even though its not,
strictly speaking, necessary.

We'll start off creating a new Phoenix project :

```
    ▶ mix phoenix.new datatables
    * creating datatables/config/config.exs
    * creating datatables/config/dev.exs
    * creating datatables/config/prod.exs
    * creating datatables/config/prod.secret.exs
    * creating datatables/config/test.exs
    * creating datatables/lib/datatables.ex
    * creating datatables/lib/datatables/endpoint.ex
    * creating datatables/test/views/error_view_test.exs
    * creating datatables/test/support/conn_case.ex
    * creating datatables/test/support/channel_case.ex
    * creating datatables/test/test_helper.exs
    * creating datatables/web/channels/user_socket.ex
    * creating datatables/web/router.ex
    * creating datatables/web/views/error_view.ex
    * creating datatables/web/web.ex
    * creating datatables/mix.exs
    * creating datatables/README.md
    * creating datatables/web/gettext.ex
    * creating datatables/priv/gettext/errors.pot
    * creating datatables/priv/gettext/en/LC_MESSAGES/errors.po
    * creating datatables/web/views/error_helpers.ex
    * creating datatables/lib/datatables/repo.ex
    * creating datatables/test/support/model_case.ex
    * creating datatables/priv/repo/seeds.exs
    * creating datatables/.gitignore
    * creating datatables/brunch-config.js
    * creating datatables/package.json
    * creating datatables/web/static/css/app.css
    * creating datatables/web/static/css/phoenix.css
    * creating datatables/web/static/js/app.js
    * creating datatables/web/static/js/socket.js
    * creating datatables/web/static/assets/robots.txt
    * creating datatables/web/static/assets/images/phoenix.png
    * creating datatables/web/static/assets/favicon.ico
    * creating datatables/test/controllers/page_controller_test.exs
    * creating datatables/test/views/layout_view_test.exs
    * creating datatables/test/views/page_view_test.exs
    * creating datatables/web/controllers/page_controller.ex
    * creating datatables/web/templates/layout/app.html.eex
    * creating datatables/web/templates/page/index.html.eex
    * creating datatables/web/views/layout_view.ex
    * creating datatables/web/views/page_view.ex

    Fetch and install dependencies? [Yn] Y
    * running mix deps.get
    * running npm install && node node_modules/brunch/bin/brunch build
```

    We are all set! Run your Phoenix application:

```
        $ cd datatables
        $ mix phoenix.server
```

    You can also run your app inside IEx (Interactive Elixir) as:

```
        $ iex -S mix phoenix.server
```

    Before moving on, configure your database in config/dev.exs and run:

```
        $ mix ecto.create
```

In order to demonstrate server side paging it's good to have a fairly large
dataset. Luckily there are numerous sources of data that can be used. For this
example I used zipcodes. I randomly choose one that I found when searching.
Its available at https://github.com/devongovett/zipcode. The file consists of
the following fields : zipcode, city, state_abbreviation. Not very interesting
but its fine to demonstrate the idea.

Lets just create a migration for our table we'll use for the zipcodes.

```
    ▶ mix ecto.gen.migration create_zips
    * creating priv/repo/migrations
    * creating priv/repo/migrations/20170212204915_create_zips.exs
```

This created the file 20170212204915_create_zips.exs but left it empty. You
have to provide the information (in this case, creating a table). Edit the file to give :

```
    defmodule Datatables.Repo.Migrations.CreateZips do
      use Ecto.Migration

      def change do
        create table(:zips) do
          add :zip_code,     :string
          add :city,    :string
          add :state,     :string

          timestamps()

      end
    end
```

Migrations should provide a way to migrate and a way to rollback. The change method
can be used in migrations if Ecto can puzzle out how to do a rollback. If you are
doing something more complex than creating a table or index you'll have to use the
up and down methods instead of change. The down method allows you to code your
own rollback.

We probably should have made zip_code unique but since we're just using this
table to demonstrate server-side paging I've left it as-is.

The schema object used for interacting with the database is simple.

```
    defmodule Datatables.Zip do
      use Ecto.Schema

      schema "zips" do
        field :zip_code,  :string
        field :city,      :string
        field :state,     :string

        timestamps
      end
    end
```

I cloned the github repo containing the zip_codes.csv file (with no header) then
fired up psql with :

```
    psql datatables_dev
    datatables_dev=# create table t(zipcode character varying, city character varying, state character varying);
    datatables_dev=# COPY t FROM 'zip_codes.csv' WITH DELIMITER ',';
    COPY 42741
    datatables_dev=# insert into zips SELECT nextval('zips_id_seq'), t.zipcode, t.city, t.state, now(), now() FROM t;
    INSERT 0 42741
```

So now the data is in the database table.

Since datatables relies on jquery we'll have to update the package.json file to install jQuery
and datatables. Modify the "devDependencies" section in package.json to include the following :

```
    "jquery": "^3.1.1",
    "datatables.net": "~1.10.12",
    "datatables.net-bs": "~1.10.12",
    "bootstrap": "~3.3",
```

And then run :

```
    npm install
```

A couple of more infrastructure things and we'll be ready to view the web page. In
order to do paging we need to include the Ecto pagination library - Scrivener. We
setup our Repo to use Scrivener with :

```
    defmodule Datatables.Repo do
      use Ecto.Repo, otp_app: :datatables
      use Scrivener, page_size: 25
    end
```

and we add :

```
    {:scrivener_ecto, "~> 1.0"},
```

to our mix.exs and also add :scrivener_ecto to the set of applications in mix.exs.

Now we do :

```
    ▶ mix deps.get
    Running dependency resolution...
    Dependency resolution completed:
      scrivener 2.2.1
      scrivener_ecto 1.1.3
    * Getting scrivener_ecto (Hex package)
      Checking package (https://repo.hex.pm/tarballs/scrivener_ecto-1.1.3.tar)
      Using locally cached package
    * Getting scrivener (Hex package)
      Checking package (https://repo.hex.pm/tarballs/scrivener-2.2.1.tar)
      Using locally cached package
```

Now, before we go further lets see if this works in iex. We'll try getting a page
worth of data but sort it in descending order by city name.

```
    ▶ iex -S mix
    Erlang/OTP 19 [erts-8.0.2] [source] [64-bit] [smp:8:8] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]

    Interactive Elixir (1.3.3) - press Ctrl+C to exit (type h() ENTER for help)
    iex(1)> alias Datatables.Zip
    Datatables.Zip
    iex(2)> import Ecto.Query
    Ecto.Query
    iex(3)> query = from z in Zip, order_by: [desc: z.city]
    #Ecto.Query<from z in Datatables.Zip, order_by: [desc: z.city]>
    iex(4)> results = Datatables.Repo.paginate(query)
    [debug] QUERY OK db=400.8ms
    SELECT count('1') FROM (SELECT z0."id" AS "id", z0."zip_code" AS "zip_code", z0."city" AS "city", z0."state" AS "state", z0."inserted_at" AS "inserted_at", z0."updated_at" AS "updated_at" FROM "zips" AS z0 ORDER BY z0."city" DESC) AS s0 []
    [debug] QUERY OK source="zips" db=37.7ms decode=2.8ms
    SELECT z0."id", z0."zip_code", z0."city", z0."state", z0."inserted_at", z0."updated_at" FROM "zips" AS z0 ORDER BY z0."city" DESC LIMIT $1 OFFSET $2 [25, 0]
    %Scrivener.Page{entries: [%Datatables.Zip{__meta__: #Ecto.Schema.Metadata<:loaded, "zips">,
       city: "ZWOLLE", id: 31343, inserted_at: ~N[2017-02-13 13:51:48.185440],
       state: "LA", updated_at: ~N[2017-02-13 13:51:48.185440], zip_code: "71486"},
      %Datatables.Zip{__meta__: #Ecto.Schema.Metadata<:loaded, "zips">,
```

So, yep..it looks like it works. If you inspect the results variable (with "i results") you'll see the
following information about it.

```
    Data type
      Scrivener.Page
    Description
      This is a struct. Structs are maps with a __struct__ key.
    Reference modules
      Scrivener.Page, Map
```

So, its a struct containing some paging information. It contains an entries element that
contains the data that we asked for and various paging related elements. The paging
elements in the struct are :

```
    page_number: 1, page_size: 25, total_entries: 42741, total_pages: 1710
```

You can see that although we didn't pass any information into the paginate call
we received the 25 rows specified as a default on our Repo. By default you'll get
the first page of rows.

Let's build a route and a controller now. Datatables.js server-side paging works by
presenting a web page and then firing an API call to the server in order to get data
to populate the table so we'll need both a regular "/" (browser) scope and an "api" scope.

```
    scope "/", Datatables do
      pipe_through :browser # Use the default browser stack
      resources "/zips", ZipController, only: [:index]

      get "/", PageController, :index
    end

    # Other scopes may use custom stacks.
    scope "/api", Datatables do
      pipe_through :api

      resources "/zips", ZipApiController, only: [:index]
    end
```

Now lets create the controllers, the server-side template to get our initial page
and the javascript that turns our table into a "datatable".

The ZipController is incredibly simple. We're not doing any heavy lifting in that
code. Simply displaying the page.

```
    defmodule Datatables.ZipController do
      use Datatables.Web, :controller

      def index(conn, _params) do
        render(conn, "index.html")
      end
    end
```

The ZipApiController has the logic, such as it is. First, we need to consider that
datatables.js is what is performing the AJAX request and its going to send parameters
in that make sense to it. Its up to the server to interpret this data. For our
purposes we'll have to map these datatables parameters into data that can be
understood by Scrivener. So lets write a simple module to help with parsing what
datatables sends up :

```
    # Provide common code for parsing paging parameters
    defmodule Datatables.DatatablesParamParser do
      def build_paging_info(params) do
        page_size = calculate_page_size(params["length"])
        page_number = calculate_page_number(params["start"], page_size)
        search_term = params["search"]["value"]
        draw_number = increment_draw(params["draw"])
        {page_size, page_number, draw_number, search_term}
      end

      defp increment_draw(value) when value == nil, do: 1
      defp increment_draw(value) do
        {draw_number, _} = Integer.parse(value)
        draw_number + 1
      end

      defp calculate_page_number(nil, _), do: 1
      defp calculate_page_number(value, page_size) do
        {start_value, _} = Integer.parse(value)
        round((start_value / page_size) + 1)
      end

      defp calculate_page_size(nil), do: 25
      defp calculate_page_size(value) do
        {page_size, _} = Integer.parse(value)
        page_size
      end
    end
```

The datatables "draw_number" took a bit to figure out. Datatables wants that
number to increment each time it gets data so it knows that it doesn't overwrite
new data with old data. The "search_term" in the parameters is the global search
for datatables. We're not going to provide individual search per column. We will
just let the global search do a search across zip_code and city.

With this in place our ZipApiController becomes :

```
    defmodule Datatables.ZipApiController do
      use Datatables.Web, :controller
      alias Datatables.Zip
      alias Datatables.DatatablesParamParser

      def index(conn, params) do
        {page_size, page_number, draw_number, search_term} = DatatablesParamParser.build_paging_info(params)
        zips = retrieve_zips(page_size, page_number, search_term)
        render(conn, :index,
               zips: zips,
               page_number: page_number,
               draw_number: draw_number)
      end

      defp retrieve_zips(page_size, page_number, search_term) do
        query = from z in Zip,
        select: struct(z, [:zip_code, :city, :state ])
        query = add_filter(query, search_term)
        Datatables.Repo.paginate(query, page: page_number, page_size: page_size)
      end

      defp add_filter(query, search_term) when search_term == nil or search_term == "", do: query
      defp add_filter(query, original_search_term) do
        search_term = "#{original_search_term}%"
        from z in query,
        where: like(z.zip_code, ^search_term) or like(z.city, ^search_term)
      end
    end
```

When the render call is done Phoenix will use the View associated with this route :
Datatables.ZipApiView. This module needs to implement an index function that will
render the JSON returned to the client.

```
    defmodule Datatables.ZipApiView do
      use Datatables.Web, :view

      def render("index.json", %{zips: zips, draw_number: draw_number}) do
        %{ recordsTotal:  zips.total_entries,
           draw: draw_number,
           recordsFiltered: zips.total_entries,
           data: Enum.map(zips, &zip_json/1)
         }
      end

      def zip_json(zip) do
        %{
          zip_code: zip.zip_code,
          city: zip.city,
          state: zip.state
        }
      end
    end
```

Notice that we're setting the recordsTotal, recordsFiltered, and draw JSON
elements. Those are all required by datatables.js. The data element needs to
contain the data used to populate the table.

Now, all we need is the javascript to tie this altogether. For the javascript I used
a technique that I found on a great blog post at [https://blog.diacode.com/page-specific-javascript-in-phoenix-framework-pt-1](https://blog.diacode.com/page-specific-javascript-in-phoenix-framework-pt-1).
It describes how to do page specific javascript. Not actually needed for this
simple project but I like the idea so much that I'll use it here too.

We'll first modify our LayoutView as described in the blog post above with :

```
    defmodule Datatables.LayoutView do
      use Datatables.Web, :view

      @doc """
      Generates name for the JavaScript view we want to use
      in this combination of view/template.
      """
      def js_view_name(conn, view_template) do
        [view_name(conn), template_name(view_template)]
        |> Enum.reverse
        |> List.insert_at(0, "view")
        |> Enum.map(&String.capitalize/1)
        |> Enum.reverse
        |> Enum.join("")
      end

      # Takes the resource name of the view module and removes the
      # the ending *_view* string.
      defp view_name(conn) do
        conn
        |> view_module
        |> Phoenix.Naming.resource_name
        |> String.replace("_view", "")
      end

      # Removes the extension from the template and returns
      # just the name.
      defp template_name(template) when is_binary(template) do
        template
        |> String.split(".")
        |> Enum.at(0)
      end
    end
```

Now we add to our body tag in app.html.eex

```
    <body data-js-view-name="<%= js_view_name(@conn, @view_template) %>">
```

This allows each page to have a name that can be grabbed by Javascript and from
there we can direct to the right js file to load for the page. You can check out
the Javascript in the github repo and go read the original blog post for insights
but what we actually end up with that matters for the datatables is a js file that
does the following : 1) gets datatables.net-bs into our scope and binds it with
jQuery; 2) sets up our table on that page (identified by its id) as a DataTable
with parametere that indicate that its populated with an Ajax call; 3) the js
also sets up a debug statement that will show us the JSON returned by the server.

```
    import MainView from '../main';
    import $ from "jquery"
    require( 'datatables.net-bs' )( window, $ );

    export default class View extends MainView {
      // Called after the view becomes active
      mount() {
        super.mount();

        console.log('ZipsIndexView mounted');
        var table = $('#zips').DataTable(
          { "processing": true,
            "serverSide": true,
            "pageLength": 25,
            "searching": true,
            "searchDelay": 1500,
            "ordering": false,
            "lengthChange": false,
            "ajax": {
               "url": "/api" + window.location.pathname
            },
            "columns": [
                { "data": "zip_code" },
                { "data": "city" },
                { "data": "state" }
            ]
          });
          table.on( 'xhr', function ( e, settings, json ) {
            console.log( 'Ajax event occurred. Returned data: ', json );
          } );
      }
      unmount() {
        super.unmount();

        // Specific logic here
        console.log('ZipsIndexView unmounted');
      }
    }
```

So, that was a number of moving parts and became a bit long-winded but its a
complete example of how to use datatables.js with Phoenix. Hopefully its helpful
to someone.
