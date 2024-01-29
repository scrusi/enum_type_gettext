# EnumType

Generates Enumerated type modules that can be used as values and matched in code. Creates proper types so Dialyzer will be able to check bad calls.

## Fork
This is an experimental fork. Please refer to the original repository unless you are specifically interested in my experiments with Gettext.

## Why?

Something we have often wanted in Elixir was an enumerated type. The benefits we wanted are:

 - compile time type checking
 - easy pattern matching
 - not Ecto specific, but has Ecto support

## Installation

The package can be installed
by adding `enum_type` to your list of dependencies in `mix.exs`:

```elixir
def deps do
  [
    {:enum_type, "~> 1.1.0"}
  ]
end
```

## Changelog

 - `v1.1.3` Replace `use Ecto.Type` with default implementation of `equal?/2`
   and `embed_as/1`.
 - `v1.1.2` Replace `@behaviour Ecto.Type` with `use Ecto.Type`
 - `v1.1.1` Fix warning for Elixir 1.10
 - `v1.1.0` Adds types to the enum. Dialyzer might start complaining.

## Creating and using an Enum

An enum type is created as its own module with each value of an an enum being a child module. The enum reference can then be used
as you would use any module name in Elixir. Since actual modules are created, this also means the module names are valid references
that can be called. Enum types can be defined anywhere `defmodule` can be used.

```elixir
defmodule MyApp do
  use EnumType

  defenum Color do
    value Red, "red"
    value Blue, "blue"
    value Green, "green"

    default Blue
  end

  @spec do_something(color :: Color.t) :: String.t
  def do_something(Color.Red), do: "got red"
  def do_something(Color.Blue), do: "got blue"
  def do_something(Color.Green), do: "got green"
end

MyApp.Color.Blue == MyApp.Color.default
"green" == MyApp.Color.Green.value
"got red" == MyApp.do_something(MyApp.Color.Red)
```

## Enum Type Functions

* `values`              - List of all enum values in the order they are defined. `["red", "blue"]`
* `enums`               - List of all enum value modules that are defined in the order defined. `[MyApp.Color.Red, MyApp.Color.Blue]`
* `options`             - List of tuples with the module name and the value. `[{MyApp.Color.Red, "red}, {MyApp.Color.Blue, "blue"}]`
* `from`                - Converts a value to an option module name. `MyApp.Color.Red == MyApp.Color.from("red")`
* `value`               - Converts a option module into the value. `"red" == MyApp.Color.value(MyApp.Color.Red)`

## Enum Option Functions

* `value`               - The value of the enum option. `MyApp.Color.Red.value`

## Custom Functions

Since both the enum type and options are both modules, any custom code that can be added to a module can also be added to these code blocks.

```elixir
import EnumType

defenum Color do
  value Red, "red" do
    def statement, do: "I'm red"
  end

  value Blue, "blue" do
    def statement, do: "I'm blue"
  end

  value Green, "green" do
    def statement, do: "I'm green"
  end

  default Blue

  @spec do_something(color :: Color.t)
  def do_something(Color.Red), do: "got red"
  def do_something(Color.Blue), do: "got blue"
  def do_something(Color.Green), do: "got green"

  def statement(color), do: "I'm #{color.value}"
end

"got blue" == Color.do_something(Color.Blue)
"I'm green" == Color.Green.statement
"I'm red" == Color.statement(Color.Red)
```

## Ecto Type Support

If `Ecto` is included in your project, additional helpers functions will be compiled in that implement the `Ecto.Type` behaviour callbacks.
When using Ecto, a type must be specified for the enum values that is supported by Ecto. All values provided by the Enum Type must be the same
Ecto basic type defined. By default, the type is `:string`.

```elixir
defmodule Subscriber do
  use Ecto.Schema
  use EnumType

  import Ecto.Changeset

  # For database field defined as a string.
  defenum Level do
    value Basic, "basic"
    value Premium, "premium"

    default Basic
  end

  # For database field defined as an integer.
  defenum AgeGroup, :integer do
    value Minor, 0
    value Adult, 1
    value NotSpecified, 2

    default NotSpecified
  end

  schema "subscribers" do
    field :name,            :string
    field :level,           Level,        default: Level.default
    field :age_group,       AgeGroup,     default: AgeGroup.default
  end

  changeset(schema, params) do
    schema
    |> cast(params, [:name, :level, :age_group])
    |> Level.validate(:level, message: "Invalid subscriber type")
    |> AgeGroup.validate(:age_group)
  end
end
```

When working with an enum type in an Ecto schema, always use the module name of the value you wish to use. The name can
be included in any query or changeset params.

## Using with Absinthe

Absinthe provides a means to define enums and map to other values. When using Absinthe with an Ecto schema that also uses `EnumType`,
you will need to map to the enum option module name and not the underlying value that will be stored in the database.

```elixir
enum :subscriber_level do
  value :basic, as: Subscriber.Level.Basic
  value :premium, as: Subscriber.Level.Premium
end

object :subscriber do
  field :level, :subscriber_level
end
```

Absinthe will produce an upper case value based upon its own enum through the graphql interface. "BASIC" or "PREMIUM".

Outbound or inbound will be mapped correctly to and from the EnumType module name value.

## Dialyzer

Enum types and values are modules under the hood. Since module names are just atoms in Elixir, we don't get compile-time type checks. However, Dialyzer will be able to spot type errors. For example, code like the following:

```elixir
defenum MyEnum do
    value One, "one"
    value Two, "two"
    value Three, "three"
end

@spec foo(thing :: MyEnum.t()) :: atom()
def foo(MyEnum.One), do: :one_here
def foo(MyEnum.Two), do: :two_here
def foo(MyEnum.Three), do: :three_here

def bad_func() do
  foo(MyEnum.NotASubtype)
end
```

Dialyzer will complain that `bad_func()` doesn't return because `foo(MyEnum.NotASubtype)` breaks the contract.
