---
date: "2017-10-15T10:07:32Z"
title: Elixir Macros Part 2 - How to find out what's going on
url: /elixir-macros-2
---

In [part 1]({% post_url 2017-10-03-elixir-macros-part1 %}), we looked into how Elixir macros work and how they allow us to **write code that writes code**, as many say.

One drawback of using macros is that they are hiding complexity, making it less obvious what exactly a given piece of code is doing. Today, we'll figure out how to dive into libraries and track down what macros are doing under the hood.

To make this as realistic as possible, we'll use an example from real library ([`Plug`](https://hexdocs.pm/plug/readme.html)). In a nutshell, `Plug` provides abstractions to deal with HTTP requests and responding to them.

## Whirlwind introduction to Plug

To understand what `Plug` does, you just need to understand two concepts.
Firstly, the concept of a `Plug.Conn` to represent an incoming request being processed, and secondly the `Plug` itself, which represents a processing step on a `Plug.Conn`.

#### The Conn data structure

When you consider a HTTP request entering your application, there is a bunch of data that you care about. Starting from the HTTP verb (e.g., `GET` / `POST`), over HTTP headers (e.g. `Authorization` to check which user made the request), to the request body.

[`%Plug.Conn{}`](https://hexdocs.pm/plug/Plug.Conn.html) is an Elixir struct that encapsulates all this data.
Processing of requests is then done by routing this data structure through a sequence of processing steps (those steps are called `Plugs`), fabricating the response.

#### Plugs as a way of transforming the Conn

As we just touched on, the `%Plug.Conn{}` also contains data about how the HTTP response should look like, e.g., the HTTP response code and response body.

A `Plug` is literally just a definition of a transformation step, either in module form (`ModulePlug.call(conn, opts)`) or function form (`functionPlug(conn, opts))`).

Now let's imagine a MVC scenario where an incoming request hits a router, gets dispatched to a controller action and eventually is answered by the server, maybe with a rendered HTML view or JSON response.

Conceptually, the running web server implementation will receive the HTTP request, wrap it into its `%Plug.Conn{}` representation, and pass it to the router implementation given in it's configuration.

Now for the really interesting part: This router is a `Plug`. It receives a `%Plug.Conn{}`, transforms it and returns a `%Plug.Conn{}`. That's all that `Plug`s ever do.
Calling our router might look as simple as:

{{< highlight elixir >}}
AppRouter.call(conn, additional_router_options)
{{< /highlight >}}

The router may then, as a part of its processing, match on the request's HTTP verb and path and delegate to a controller. Guess what that controller is. Yuuuuuup, surprise, it's a `Plug`.

Congratulations, you now know everything there is to know about `Plug`.

## Deconstructing the Plug router

Here's a very simple router definition (borrowed straight from [the excellent `Plug.Router` documentation](https://hexdocs.pm/plug/Plug.Router.html)).

{{< highlight elixir >}}
defmodule AppRouter do
  use Plug.Router

  plug :match
  plug :dispatch

  get "/hello" do
    send_resp(conn, 200, "world")
  end

  match _ do
    send_resp(conn, 404, "oops")
  end
end
{{< /highlight >}}

I won't insult your intelligence by reiterating what this router does.
Rather, I'll tell you how to figure out how this code functions.

#### What's that `plug` thingie and how does it exactly work?

At the time of typing these words, _I don't know either_. Let's find out.

To start, we might wonder if the `plug` keyword is part of Elixir itself (**Spoiler**: It's not.). Excellent. Apparently this is a macro. The next question we might ask is: "Where is this coming from?"

Generally, macros cannot just appear out of thin air, Elixir doesn't allow for this, because it's intransparent.

The only two ways that macros may enter a module is as follows:

- **require**: \\
  This allows us to use macros defined in the `require`d module in the form of:
{{< highlight elixir >}}
require Logger

Logger.info "Hello world!" # `info` is a macro defined in the `Logger` module
{{< /highlight >}}

- **import**: \\
  This merges the `import`ed module's functions and macros into the scope of the current module.
  We can then call macros like:
{{< highlight elixir >}}
import Logger

info "Hello world!" # Note how we don't need the `Logger` prefix!
{{< /highlight >}}

- **use**:
  This is our special candidate. It does not directly make macros available to our module, but it can include `require` or `import` statements!

  **Super tl;dr version of what this statement does:**
  It invokes the `__using__` macro in the `use`d module, so its just the syntactic sugar equivalent to `TheModule.__using__`. Mostly used to dump some functionality into your module (Notice how we're using `Plug.Router` above!)

Having established what to look for when hunting macros, let's go back to our router example. We were trying to figure out where the `plug` macro is coming from, remember?

The only line that may inject macros is obviously:

{{< highlight elixir >}}
use Plug.Router
{{< /highlight >}}

Diving into the [`Plug.Router` source code](https://github.com/elixir-plug/plug/blob/v1.4.3/lib/plug/router.ex#L202), we find:

{{< highlight elixir >}}
# Some details omitted for brevity
defmacro __using__(opts) do
  quote location: :keep do
    import Plug.Router

    use Plug.Builder, unquote(opts)

    def match(conn, _opts) do
      do_match(conn, conn.method, Enum.map(conn.path_info, &URI.decode/1), conn.host)
    end

    def dispatch(%Plug.Conn{assigns: assigns} = conn, _opts) do
      Map.get(conn.private, :plug_route).(conn)
    end

    defoverridable [match: 2, dispatch: 2]
  end
end
{{< /highlight >}}

Hmm, no `defmacro plug`... OK. Repeat the process. We got two new candidates!
1. `import Plug.Router`
2. `use Plug.Builder`

Glancing over the [functions in `Plug.Router`](https://hexdocs.pm/plug/Plug.Router.html#functions), no hits. Moving on.

Checking on [`Plug.Builder.__using__`](https://github.com/elixir-plug/plug/blob/v1.4.3/lib/plug/builder.ex#L102):

<a id="builder_using"></a>
{{< highlight elixir >}}
defmacro __using__(opts) do
  quote do
    # Again, shortened for clarity

    def init(opts) do
      opts
    end

    def call(conn, opts) do
      plug_builder_call(conn, opts)
    end

    import Plug.Conn
    import Plug.Builder, only: [plug: 1, plug: 2] # <- OHHHHHHHH BABY

    Module.register_attribute(__MODULE__, :plugs, accumulate: true)
    @before_compile Plug.Builder
  end
end
{{< /highlight >}}

**Jackpot!** So `plug` is defined in [`Plug.Builder`](https://hexdocs.pm/plug/Plug.Builder.html)!

Go deeper you say? Hell yeah! On to the [`plug` macro definition](https://github.com/elixir-plug/plug/blob/v1.4.3/lib/plug/builder.ex#L150).
{{< highlight elixir >}}
defmacro plug(plug, opts \\ []) do
  quote do
    @plugs {unquote(plug), unquote(opts), true}
  end
end
{{< /highlight >}}

Apparently, calling `plug` just sets the `@plugs` module attribute to a 3-item tuple. So what's that about?

We could technically wrap up here, we found the macro definition!

<div style="width:100%;height:0;padding-bottom:56%;position:relative;"><iframe src="https://giphy.com/embed/a0h7sAqON67nO" width="100%" height="100%" style="position:absolute" frameBorder="0" class="giphy-embed" allowFullScreen></iframe></div><p><a href="https://giphy.com/gifs/borat-great-success-a0h7sAqON67nO">via GIPHY</a></p>

But where's the fun in that?

## How Plugs can be composed out of other Plugs

I'd like you to go back to <a href="#builder_using">the `Plug.Builder.__using__` macro</a> for a second.

Notice the two last lines:
{{< highlight elixir >}}
Module.register_attribute(__MODULE__, :plugs, accumulate: true)
@before_compile Plug.Builder
{{< /highlight >}}

To shorten this a little bit, the call to [`Module.register_attribute`](https://hexdocs.pm/elixir/Module.html#register_attribute/3) states that we want our module to not reassign `@plugs` when we repeatedly assign it but instead add the new values to an accumulating list.

The `@before_compile` module attribute is reserved by the language and hints the Elixir compiler to invoke the `__before_compile__/1` macro in `Plug.Builder` before compiling the module and put its output at the end of the router module definition ([for curious people](https://hexdocs.pm/elixir/Module.html#module-compile-callbacks)).

Here we go, what's in the [`__before_compile__` box](https://github.com/elixir-plug/plug/blob/v1.4.3/lib/plug/builder.ex#L126)?

{{< highlight elixir >}}
defmacro __before_compile__(env) do
  # Apparently we're grabbing all the defined plugs from the @plugs list
  plugs        = Module.get_attribute(env.module, :plugs)
  builder_opts = Module.get_attribute(env.module, :plug_builder_opts)

  # Oh look, it's compiling all our plugs into something new!
  {conn, body} = Plug.Builder.compile(env, plugs, builder_opts)

  quote do
    # Defines a private function on our router module, not too bad
    defp plug_builder_call(unquote(conn), _), do: unquote(body)
  end
end
{{< /highlight >}}

OK, everything is straightforward. Just `Plug.Builder.compile/3` seems crazy, fasten your seatbelt, we're [going in](https://github.com/elixir-plug/plug/blob/v1.4.3/lib/plug/builder.ex#L178)!

{{< highlight elixir >}}
@spec compile(Macro.Env.t, [{plug, Plug.opts, Macro.t}], Keyword.t) :: {Macro.t, Macro.t}
def compile(env, pipeline, builder_opts) do
  conn = quote do: conn
  {conn, Enum.reduce(pipeline, conn, &quote_plug(init_plug(&1), &2, env, builder_opts))}
end
{{< /highlight >}}

To keep this post somewhat manageable, we are going to handwave just the tiniest bit and just give you the big picture.

Our router module calls the <a href="#builder_using">`Plug.Builder.__using__` macro</a>, which defines `call/2` on it (routers are `Plug`s, remember?), which just delegates to `plug_builder_call`.

To define `plug_builder_call`, we are running over all the `Plug`s filled into the `@plugs` module attribute (e.g, using the `plug` macro that we examined) - those are in the `pipeline` parameter of `Plug.Builder.compile`.

From there, we are just nesting the `Plug`s in one another, effectively ending up with something along the lines of this:

{{< highlight elixir >}}
defmodule AppRouter do
  def call(conn, opts) do
    plug_builder_call(conn, opts)
  end

  defp plug_builder_call(conn, opts)
    conn
    |> match(opts)
    |> dispatch(opts)
  end
end
{{< /highlight >}}

The `call/2` function now just composes the other `Plug`s defined within it, in the order of appearance. Neat, huh? It's like a chain of transfomation steps that our request is handed through!

The `match` and `dispatch` function plugs are defined in `Plug.Router.__using__`, but this post is already 1400 words long, so we'll wrap it up here :-)

Until next time!
