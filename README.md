# Filtrex

[![Hex.pm](https://img.shields.io/hexpm/v/filtrex.svg)](https://hex.pm/packages/filtrex)
[![Build Status](https://travis-ci.org/rcdilorenzo/filtrex.svg?branch=master)](https://travis-ci.org/rcdilorenzo/filtrex)
[![Docs Status](http://inch-ci.org/github/rcdilorenzo/filtrex.svg?branch=master)](http://inch-ci.org/github/rcdilorenzo/filtrex)

So often, filtering from URL parameters or from some client description of a "smart" filter can be extremely tedious. This library is an attempt to address that problem by flexibly converting either URL parameters or a parsed JSON body to a consistent filter structure and even straight to an `Ecto` query. It also supports validation of both allowed keys and their value types with configuration options specific to that type (e.g. allowing a decimal point in a number filter or what format is allowed for dates).

Filtrex is an elixir library for parsing and querying with filter data structures and parameters. It allows the construction of Ecto queries from Phoenix-like query parameters or map data structures for saving "smart" filters. It has been tested using the Postgres adapter but will potentially work with other adapters as well.

Feel free to check out the [published docs](https://hexdocs.pm/filtrex/) for the latest and greatest information.


## Parsing Filters from URL Params

Here's the rough outline of how to use the parameter parsing capabilities of Filtrex. Config options presented are discussed in further detail later in this README. Also, see below for an [example using Phoenix](#params-filter-example-with-phoenix).

```elixir
# Get params from phoenix controller (or anywhere else)
params = %{
    "comments_contains" => "Chris McCord",
    "title" => "Upcoming Phoenix Features",
    "posted_at_between" => %{"start" => "01-01-2013", "end" => "12-31-2017"},
    "filter_union" => "any"  # special value for filter type (any | all | none)
}

# Create validation options for keys and formats
config = [
  %Filtrex.Type.Config{type: :text, keys: ~w(title comments)},
  %Filtrex.Type.Config{type: :date, keys: ~w(posted_at), options: %{format: "{0M}-{0D}-{YYYY}"}}
]

# Parse params into encodable filter structures
{:ok, filter} = Filtrex.parse_params(config, params)

# Encode filter structure into where clause on Ecto query
query = from(s in YourApp.YourModel)
  |> Filtrex.query(filter)  # => #Ecto.Query<...
```

Using parsed parameters from your phoenix application, a filter can be easily constructed with type validation and custom comparators.

## Parsing Filter Structures

```elixir
# Create validation options for keys and formats
config = [
  %Filtrex.Type.Config{type: :text, keys: ~w(title comments)},
  %Filtrex.Type.Config{type: :date, keys: ~w(due_date)},
  %Filtrex.Type.Config{type: :boolean, keys: ~w(flag)}
]

# Parse a "smart-filter" encoded from client (e.g. with Poison or Phoenix)
{:ok, filter} = Filtrex.parse(config, %{
  "filter" => %{
    "type" => "all",               # all | any | none
    "conditions" => [
      %{"column" => "title", "comparator" => "contains", "value" => "Buy", "type" => "text"},
      %{"column" => "title", "comparator" => "does not contain", "value" => "Milk", "type" => "text"},
      %{"column" => "flag", "comparator" => "equals", "value" => "false", "type" => "boolean"}
    ],
    "sub_filters" => [%{
      "filter" => %{
        "type" => "any",
        "conditions" => [
          %{"column" => "due_date", "comparator" => "equals", "value" => "2016-03-26", "type" => "date"}
        ]
      }
    }]
  }
})

# Encode filter structure into where clause on Ecto query
query = from(m in YourApp.YourModel, where: m.rating > 90)
  |> Filtrex.query(filter)  # => #Ecto.Query<...

```

The configuration passed into `Filtrex.parse/2` gives the individual condition types more information to validate the filter against and is a required argument. See [this section](http://rcdilorenzo.github.io/filtrex/Filtrex.html) of the documentation for details. The entire [documentation](http://rcdilorenzo.github.io/filtrex) is filled with valuable information on how to both use and extend the library to your liking so please take a look!

## Params Filter Example with Phoenix

In `some_controller.ex`:
```elixir
import Ecto.Query

def index(conn, params = %{"user_id" => user_id}) do
  filter_params = Map.drop(params, ~w(user_id))
  base_query = from(m in Model, m.user_id == ^user_id)
  case Filtrex.parse_params(Model.filter_options(:admin), filter_params) do
    {:ok, filter} ->
      render conn, "index.json", data: Filtrex.query(base_query, filter) |> Repo.all
    {:errors, errors} ->
      render conn, "errors.json", data: errors
  end
end
```

In `model.ex`:
```elixir
alias Filtrex.Type.Config

def filter_options(:admin) do
  [%Config{type: :text, keys: ~w(title)},
   %Config{type: :date, keys: ~w(published)},
   %Config{type: :number, keys: ~w(upvotes)},
   %Config{type: :boolean, keys: ~w(flag)},
   %Config{type: :datetime, keys: ~w(updated_at inserted_at)},
   %Config{type: :number, keys: ~w(rating), options: %{allow_decimal: true}}]
end
```

With this example, below is a sample URL query that could be made:

```elixir
%{
  "title_contains" => "Conf",
  "published_on_or_before" => "2016-04-01",
  "upvotes_greater_than" => 45,
  "flag" => true,
  "updated_at_after" => "2016-04-01T12:34:56Z",
  "rating_greater_than_or" => 90.5,

  "filter_union" => "any"
}
```

## Filter Types

The following condition types and comparators are supported.

* [Filtrex.Condition.Boolean](http://rcdilorenzo.github.io/filtrex/Filtrex.Condition.Boolean.html)
    * equals, does not equal
* [Filtrex.Condition.Text](http://rcdilorenzo.github.io/filtrex/Filtrex.Condition.Text.html)
    * equals, does not equal, contains, does not contain
* [Filtrex.Condition.Date](http://rcdilorenzo.github.io/filtrex/Filtrex.Condition.Date.html)
    * after, on or after, before, on or before, between, not between, equals, does not equal
    * options: format (default: `{YYYY}-{0M}-{0D}`)
* [Filtrex.Condition.DateTime](http://rcdilorenzo.github.io/filtrex/Filtrex.Condition.DateTime.html)
    * after, on or after, before, on or before, equals, does not equal
    * options: format (default: `{ISOz}`)
* [Filtrex.Condition.Number](http://rcdilorenzo.github.io/filtrex/Filtrex.Condition.Number.html)
    * equals, does not equal, greater than, less than or, greater than or, less than
    * options: allow_decimal (default: false), allowed_values (default: nil)

## Installation

The package is available on [Hex](https://hex.pm) and can be installed by adding it to your list of dependencies:

```elixir
def deps do
  [{:filtrex, "~> 0.3.0"}]
end
```


## License

Copyright (c) 2015-2017 Christian Di Lorenzo

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
