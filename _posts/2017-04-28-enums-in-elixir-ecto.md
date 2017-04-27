---
layout: post
title:  "Using Enums (like Rails) in Elixir"
excerpt: "Ecto in Elixir does not support Enums which is heavily used in Rails. "
date:   2017-04-28 02:14:00 +0800
categories: [elixir]
tags: [elixir,ecto]
author: Taher Dhilawala
---

We recently started working with Elixir (we are also [hiring](https://boards.greenhouse.io/honestbee/jobs/669258#.WQI1OomGNE4) for that btw :)). Most of the projects in honestbee are done in Rails and so majority of us here are Rubyist. When I first started working with Elixir, almost all of initial Google searches where like "Rails X equivalent in Elixir". To my surprise, there are a lot of people like me :).

## Phoenix

We use [Phoenix Framework](http://www.phoenixframework.org/) for developing REST APIs and web applications with Elixir.

## Ecto

[Ecto](https://github.com/elixir-ecto/ecto) is to Phoenix what Active Record is to Rails. I was very impressed by its Query API and syntax. If you have worked with .NET LINQ you would find Ecto very similar.

As I started developing my database models, very quickly I started missing `enums`. In Rails, we can easily do something like

```
class User
  enum status: {active: 1, inactive: 2}
end
```

and then I can use the value for user statuses with

```
User.statuses[:active] # this will give 1
User.statuses[:inactive] # this will give 2
```

Sadly, Ecto does not have anything similar to enum. I find it very useful to abstract the id values with meaningful names as it allows me to change the values later without affecting my code implementation.

## Macros

In Elixir, you can define a macro which will expand the code in place where you use them.  

Before, we take a look at how to write a macro, we need to understand `quote` and `unquote`

### Quoting

**Everything in Elixir is represented as a tuple with three elements**. For example, the following function `print("hello world")` is represented as

```
{:print, [], ["hello world"]}
```

First element is the function name, second is the keyword list containing metadata and third contains the arguments.

You can get this representation by doing (in your `iex` console)

```
iex> quote do: print("hello world")
{:print, [], ["hello world"]}
```

To get your string representation back from a quoted expression, you can do

```
iex> Macro.to_string({:print, [], ["hello world"]})
"print(\"hello world\")"
```

### Unquoting

Sometimes, you may want to use a variable value in your quoted expression, for example,

```
iex> user = "Taher"
iex> quote do: print("hello " <> user)
{:print, [], [{:<>, [context: Elixir, import: Kernel], ["hello ", {:user, [], Elixir}]}]}
```

Notice how we got `{:user, [], Elixir}` instead of the value of that variable. This is where `unquote` is useful.

```
iex> user = "Taher"
iex> quote do: print("hello " <> unquote(user))
{:print, [], [{:<>, [context: Elixir, import: Kernel], ["hello ", "Taher"]}]}
```

Now we have the value in the quoted expression which is what we want.

### Enum macro

To create a `enum` in elixir, I created a macro, that would take the name of the enum and a block which will contain the values (a map)

```
defmacro enum(name, [do: block]) do
end
```

The above macro can be used as

```
enum "status" do
  %{
    active: 1,
    inactive: 2
  }
end
```

Now, I need a way to read the enum values i.e. be able to do something like

```
MyModel.status[:active] # this should give me 1
```

To achieve this, we can create a method with the enum name which returns the map. Since our method will be in the macro, when we do `import MyEnumMacro` in the `MyModel` module, it will expand in place and thus will be available as a method in the module.

A macro needs to be a quoted expression so that it can be expanded by the compiler when we do import. So lets do some quoting-unquoting, shall we?

```
defmacro enum(name, [do: block]) do
  enum_values = <Fetch values from block. More on this later>
  quote do
    def unquote(:"{name}")() do # unquoting because we want to use the actual value of "name"
      unquote(enum_values)
    end
  end
end
```

What we did here is, created a quoted expression which contains a function with enum name and it returns the enum value map.

Lets get he enum values from the do block

```
enum_values =
  case block do
    {_, _, values} when is_list(values) ->
      values
    _ ->
      quote do
        {:error, "Only maps allowed as values of enum"}
      end
  end
```

If the case statement above looks alien, then read up a bit about [case](http://elixir-lang.org/getting-started/case-cond-and-if.html#case) and [pattern matching](http://elixir-lang.org/getting-started/pattern-matching.html) in elixir.

Remember that everything in elixir is represented as a three element tuple? Our do block is passed to our macro function as a tuple with the third element as our map in a keyword list (the check in the when clause). So if the format is correct, we pattern match and fetch the value map which is used as a return value of the function. If we pass something else other than a map, the pattern matching fails and the enum value is set with an error tuple. So when you try to fetch the enum value, you will get an error.

Here is the complete code

```
defmacro enum(name, [do: block]) do
  enum_values = case block do
      {_, _, values} when is_list(values) ->
          values
      _ ->
          quote do
            {:error, "please provide Map with %{key: value} for enum"}
          end
  end

  quote do
      def unquote(:"#{name}")() do
          unquote(enum_values)
      end
  end
end
```

## Summary

I hope you find this useful if you are working with Ecto and Elixir. I find Elixir very refreshing  and exciting. If you are working with Ruby and are interested in making the switch, read up some discussions about progression from Ruby to Elixir [here - quora](https://www.quora.com/Will-Elixir-Phoenix-destroy-Ruby-on-Rails) and [here - reddit](https://www.reddit.com/r/ruby/comments/39b1l1/we_all_are_going_to_meet_in_the_elixir_world/).
