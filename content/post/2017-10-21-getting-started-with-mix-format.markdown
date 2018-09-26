---
categories:
- elixir
date: "2017-10-21T10:13:25Z"
title: Getting started with mix format
---

## What's code formatting and why do we care?

On my past job, I had the pleasure of writing quite a bit of [Elm](http://elm-lang.org/) (which has the most outstanding developer experience that I've ever seen and is going to take over the frontend eventually, I hope).

One feature about the Elm workflow that really stood out to me, is that you:
- Write some code that you think should work
- Hit __Save__
- Suddenly your reedit-problem-solving-jumbled code looks perfectly fine!

This does not sound like a big deal at first, but I invite you to try it for a day and see if you ever want to go back. And in my understanding, it's not even about saved keystrokes, but about the mental context switches you're not making.

**OK, that works, time to make it looks nice, and then I can move on with the next step**

Just imagine deleting this step from the equation. If you are a developer, I don't think I have to argue to you that suspending your current train of thought does not help with solving the problem at hand. Therefore, I believe it won't be hard for you to agree that reindenting lines is not what makes you happy or valuable to the world.

## Code formatting in Elixir

Fortunately, José Valim and the amazing Elixir core team seem to agree with me and made code formatting a first-class citizen in the upcoming release 1.6 of Elixir.

At the time of writing this post, version 1.6 is not out yet.
If you want to try it today, I recommend just running the `master` branch of Elixir.

#### Setup

For all Linux and MacOS users, I recommend using [the ASDF version manager](https://github.com/asdf-vm/asdf) and it's excellent [Elixir plugin](https://github.com/asdf-vm/asdf-elixir).

From there, you can run:
{{< highlight bash >}}
# Install the current master version of Elixir, to update just reinstall it
asdf install elixir master-otp-20

# Use edge Elixir as the global default
asdf global elixir master-otp-20

# Or just for the project that you're currently in
asdf local elixir master-otp-20
{{< /highlight >}}

And have access to `mix format`, which runs the code formatter.

### Format-on-save

In my book, this is the most important use case and the setup on your editor of choice is going to vary, but any self-respecting editor has some way of running commands on save or on a keyboard shortcut.

There's even some plugins that take that configuration from you:
- [Atom](https://github.com/rgreenjr/atom-elixir-formatter)
- [Neoformat for VIM](https://github.com/sbdchd/neoformat)

**Side note** If you happen to set this up by yourself, most formatting frameworks support piping the file contents to the formatter via `STDIN` and reading the formatted output from `STDOUT`, which makes for snappier formatting. This is supported by `mix format -`.

### Whole-project formatting

To do this, you can either pass a pattern like:
{{< highlight bash >}}
mix format "lib/**/*.{ex,exs}"
{{< /highlight >}}

However, I'd recommend adding a `.formatter.exs` file to your project root (that's where `mix format` is looking for it by default, but you can put it wherever you want. That's what she said.):
{{< highlight elixir >}}
[
  inputs: [
    "mix.exs",
    "{config,lib,test}/**/*.{ex,exs}"
  ]
]
{{< /highlight >}}

This enables you to just run `mix format` with no arguments to reformat all the code in the specified paths.

That's all I got for now, hopefully my little essay persuaded you to give automated code formatting a try, if not in Elixir, then maybe in your language of choice. Until next time.
