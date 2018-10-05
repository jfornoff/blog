---
date: "2017-10-03T11:59:53Z"
title: Elixir Macros Part 1 - How they work
url: /elixir-macros-1
tags:
- elixir
---

**TL;DR** Elixir is nice. Macros enable you to create a concise language for the application problem domain without applying magic.

Today’s post will be about Elixir macros, mostly just documenting my own learning process by retelling the story.

# Why do we care?

Programming languages are used to solve all types of problems. Web servers, games, banking applications, you name it. We appreciate low-boilerplate code that caters to the problem domain. Conciseness and readability are what we’re looking for. However, it is impossible to support expressiveness for all those use cases in the core language. Therefore, we need a mechanism to create readable code.
Creating such domain-bound abstractions involves hiding frequently unimportant details. We need to put these details away while making sure that it’s very clear where to find them. This makes code maintainable and simple to reason about.

# Code as (transformable) data

Let’s take a step back. The first important idea to understand what Macros are used for is where they sit in the compilation process. When you compile Elixir code, the text of the source file is parsed into an Abstract Syntax Tree (AST), which represents the code in syntactic form.

**Simple example:**


{{< highlight elixir >}}
1 + 1
{{< / highlight >}}

is represented in the AST as:

{{< highlight elixir >}}
{:+, [context: Elixir, import: Kernel], [1, 1]}
{{< / highlight >}}

This is a tuple with 3 elements, the first being the name of the function being called, the second metadata, and the last the list of function arguments.

# What role do Macros play?

After generating the Elixir AST from the source code files, Macros are _expanded_ by the compiler.
That means: When the compiler finds a call to a Macro, the compiler executes it and substitutes the result of the Macro invocation into the AST.

Therefore, Macros need to return an AST representation, such as the one we saw in the example.
Our core takeaway from this is that Macros just return **data** (in this case a data-representation of source code).

Let's take a look at a fun little Macro.

{{< highlight elixir "linenos=inline" >}}
defmodule MyLogger do
  @enabled true

  defmacro log(message) do
    if @enabled do
      quote do
        IO.puts unquote(message)
      end
    end
  end
end
{{< / highlight >}}

This is a logger module whose `log/1` Macro you can use, and can turn off all log output using `@enabled`.

How would that work in practice?

{{< highlight elixir >}}
require MyLogger

# when @enabled is true
MyLogger.log("Hello!") # => :ok
"Hello!"

# when @enabled is false
MyLogger.log("Hello!") # => nil
{{< / highlight >}}

Note the calls to [`quote`](https://hexdocs.pm/elixir/Kernel.SpecialForms.html#quote/2) and [`unquote`](https://hexdocs.pm/elixir/Kernel.SpecialForms.html#unquote/1), those are the bread and butter of writing Macros. For the following explanations, I'd invite you to think of writing Macros as being analogous to the concept of string interpolation. The block passed `quote` outlining the string (or AST-fragment if you will), `unquote` used to insert predefined values.

Line 5 (`if @enabled`) is executed at compile time, whenever the Macro is expanded.
Therefore, if `@enabled` is false, the `quote`-block will not be returned, thus defaulting to nil and inserting nothing into the Macro-site AST.

If `@enabled` is true, `quote` states that all code inside of it will be returned as the AST data representation. The effect of this is literally as if you copy-pasted the code inside the `quote` block into the module where you called the Macro.

Now note line 7, specifically `unquote(message)`. If we did omitted `unquote()`, the message value from the function argument would've not been inserted into the AST, but instead it stays as a "placeholder" that is evaluated as soon as runtime execution passes that line. For example, it might output the result of the function `message()` of the module that we compiled the Macro into.

# Conclusion

We can conclude that expanded Macros yield some code representation that is then injected into the AST representing the Macro-calling module. When we want to inject values into that code representation (e.g, Macro function arguments), we have to use `unquote`. Otherwise the code is just executed in exactly the way we write it in `quote`.

# References

If you'd like a much longer and more in-depth resource on Elixir Macro mechanics, I highly recommend [Saša Jurić's series of blog posts on Macros](http://www.theerlangelist.com/article/macros_1)!

# Up next

Macros are constructs that make code and the origin of functionality slightly less obvious because they inject code into your modules AST. This is intended and necessary to create more concise and succinct code, but makes it more challenging to track down how things actually work.

Therefore Part 2 will be about Macro Archaeology and how to dig up functionality if you want to dive in the nitty-gritty hidden details.
