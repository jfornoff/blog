---
date: "2017-12-09T10:42:34Z"
title: Documentation Driven Development - or why names matter
---

Today's post will be a pitch for why you should be sensitive to how you refer to
concepts in your code and documentation.

First statement: [Advent of Code](http://adventofcode.com/2017) is fun as heck and you should do it.

While solving the Day 7 problem ([My solution here - Spoiler alert](https://github.com/jfornoff/advent-of-code-2017/blob/master/dayseven/lib/dayseven.ex)) I got stuck.
I knew what generally needed to happen, but couldn't quite get it right.

Everyone gets stuck, and people deal differently with this, I usually start scribbling on paper.
Today I wanted to try something different.

{{< highlight elixir >}}
@doc """
<Your mental picture of the problem>
"""
{{< /highlight >}}

### Problem solving is verbal
Especially when writing code, you need to name the concepts and abstractions you're
working with. And if you've been writing source code for an extended period of time,
you will probably agree with me when I say that there are parts of code that just "click"
when you read them.

**Yeah that's obviously doing X.**

Is it obvious though? Is it that the solved problem was just really simple that caused
you to sweep over the code without any mental friction?

I'll argue that it's not. Instead, I'll claim that the author of the code spent time
figuring out what the fragments of the problem should be called to give you the right
impression about what they're there for and where they conceptually belong.

Moving from this point, it's conclusive that you'll probably have a better time understanding
your own code *as you write it* when you pay attention to what you call things.
Every time you have to step back from writing code and wonder **Wait, is this right? What was I trying to do again?**, this will yield dividends. You can re-read where you came from and pick back up with clarity.

Now the fitting name usually won't pop into your head right away.
Here's where writing documentation (even before the code exists) can help you map out what's actually going on with the problem.

### Document to establish vocabulary
When you read through a Jira ticket or someone tells you about a problem that's been bugging them,
you subconsciously start mentally modelling the pieces of data you're dealing with
and already start sketching out what the rough solution steps would be.

If the problem has any appreciable degree of complexity, you will most likely not get it right by just eyeballing the solution approach.

This is where it helps to just write down what you conceptually want to do.

{{< highlight elixir >}}
@doc """
This parses tree nodes with their child node names from the input string and
rearranges them into a traversable Tree data structure,
represented by the root node containing its child subtrees.
"""

def build_tree(input_string) do
end
{{< /highlight >}}

From here, it is pretty straightforward to "Write the code you wish you had" (which is a saying often used in the TDD and BDD community, stating that you can **dream up** how ideal high-level code should look and not care about the underlying implementation yet)

{{< highlight elixir >}}
def build_tree(input_string) do
  input_string
  |> parse_nodes()
  |> arrange_nodes_into_tree()
end
{{< /highlight >}}

From here, you can drill down and start wondering "What do I need to do to parse this string into a list of nodes?"

Also note how the documentation-writing also eased us into establishing the core concepts of
our solution model. This is the vocabulary of your solution. **Do not deviate from it, decide!**

### Conclusion
Taking your hands off the keyboard for 10 minutes and figuring out in which terms you want to
reason about the problem at hand is not unproductive.

If you feel like the text on your screen is gibberish, try to descriptively state
what this code is trying to do, find out what's clouding the clarity and rename something for the better.

Farewell!



