---
title: Ambling Through The Ice
description: An soft introduction to Icicle
---

The Icicle language is simple and safe by design, to make it easy for data scientists, data engineers, and business intelligence practitioners to pick up and become productive quickly.

This post aims to give a gentle introduction to the concepts of Icicle, by teaching how to load data and perform simple queries interactively.

Let us start by opening the Icicle interactive repl, and loading a sample dataset. In the repl, lines starting with -- are simply comments, while lines starting with : are commands to the interpreter. Lines with neither are interpreted as Icicle expressions, as we will see soon.

```
$ icicle repl

  ██▓ ▄████▄   ██▓ ▄████▄   ██▓    ▓█████
 ▓██▒▒██▀ ▀█  ▓██▒▒██▀ ▀█  ▓██▒    ▓█   ▀
 ▒██▒▒▓█    ▄ ▒██▒▒▓█    ▄ ▒██░    ▒███
 ░██░▒▓▓▄ ▄██▒░██░▒▓▓▄ ▄██▒▒██░    ▒▓█  ▄
 ░██░▒ ▓███▀ ░░██░▒ ▓███▀ ░░██████▒░▒████▒
 ░▓  ░ ░▒ ▒  ░░▓  ░ ░▒ ▒  ░░ ▒░▓  ░░░ ▒░ ░
  ▒ ░  ░  ▒    ▒ ░  ░  ▒   ░ ░ ▒  ░ ░ ░  ░
  ▒ ░░         ▒ ░░          ░ ░ REPL ░
  ░  ░ ░       ░  ░ ░          ░  ░   ░  ░
     ░            ░
                  ░     :help for help

λ :load data/example/sample.zbin
Loaded dictionary with 1 inputs, 0 outputs, 39 functions.
Loaded 34 functions from prelude.icicle
Selected zebra binary file as input: data/example/sample.zbin
```

To find out more information about the newly loaded dictionary, we can use the `dictionary` command.

```
λ :dictionary
Dictionary
----------

## Functions

  newest : Element a -> Aggregate (Possibly a)

  oldest : Element a -> Aggregate (Possibly a)

  not : Bool -> Bool

  is_some : Option a -> Bool

  is_none : Option a -> Bool

  ...

## Inputs

  default:injury : {
      action : Option String
    , location : String
    , severity : Double
  }
```

The dictionary includes functions, such as `sum`, `sin`, and `count`, which are available for use in queries.

The inputs and outputs of a module are the most important part. The fact definitions (or inputs), display their encoding to the right of the colon. We see that injury is a structure. In Icicle, each query operates on a single stream of facts.

To find out about the environment of the icicle repl, we can use the `set` command. Which will tell us which optimisations are on, and which evaluation strategies will be used. When getting to know icicle, using commands like `:set +annotated`, to turn on the printing of a type annotated query, and `:set +core` to see the core for each query, can be very handy.

Simple Queries
------------

We can test the `mean` function, copying the following line into the REPL

```
λ from injury in mean severity
C evaluation
------------

homer|4.0
marge|4.0
moe|1.5
```

Icicle has politely introduced bindings for each field in the input structure, we can also use record syntax to look at each value with `fields.severity`.

However, finding the mean of location is not so simple: location is a string, but mean only works on numbers such as integers or doubles.

```
λ from injury in mean location
                 ^
Error
-----

## Check error

  Cannot discharge constraints at 1:16

    1:16  Not a number: String
          Chances are you tried to apply some numerical computation
          to the wrong field.
```

The important part is in the last section: `Not a number: String`.

This indicates a key point in Icicle's design. Only queries which make sense will pass the type checker; which provides a great amount of in ones work before we start deploying queries across a cluster.


Newest and oldest
-----------------

Other functions don't have the same restrictions on numbers. Consider `newest` and `oldest` which simply return the first and last entries. We can join the result of two aggregates with a comma.

```
λ from injury in newest location, oldest severity
C evaluation
------------

homer|("torso", 4.0)
marge|("head", 4.0)
moe|("hair", 1.0)

```

If we wanted the newest of both location and severity, we could use parentheses to nest the comma inside the argument to `newest` like so:

```
λ from injury in newest (location, severity)
C evaluation
------------

homer|("torso", 5.0)
marge|("head", 5.0)
moe|("hair", 2.0)

```

Filters and contexts
--------------------

Queries can be thought of as piping data through some number of transformers such as filters, then finally collecting the data in an aggregate at the end. The data must be aggregated at the end, as otherwise we might end up with an unbounded amount of output data.

The source of this data is a single concrete feature. We can imagine that `from salary` starts reading all entries from the disk. The data is then used `in` the rest of the expression. There is also a "flows into" `~>` which acts as a synonym for `in`.

```
λ from injury in mean severity
```
Here, all data is being read, and passed into the `mean` aggregation. We can introduce new "contexts" between the source and the aggregate, for example if we wanted to find the mean of injuries to the arm:

```
λ from injury in filter location == "arm" in mean severity
C evaluation
------------

homer|4.0
```

Here, the data is filtered before it reaches the aggregation. We can even split the data feed, by applying different contexts at different places, as long as the different pipelines are joined as aggregates. For example, if we wished to compute the ratio of arm to head injuries numbers, we would introduce a context `filter location == "arm"` and `filter location == "head"` like so:

```
λ from injury in (filter location == "arm" in count severity) / (filter location == "head" in count severity)
```

This gets a bit messy however, and we may wish to introduce intermediate bindings with `let`. Here we have split the query across multiple lines as one would do in a normal Icicle module, for now however, the repl only handles input queries on a single line.

```
from injury
  in let
    arm =
      filter location == "arm" in count severity
    head =
      filter location == "head" in count severity
  in
    arm / head
```

For complex queries, `let` binding become invaluable, as they allow
us to document the code and simplify our expressions.


Single-pass restriction
----------------------

What if we wanted to count the number of elements larger than the mean? In order to find the mean, we need to see all elements in the stream. To find elements in the stream larger than the mean, we would again need see all elements in the stream. This would require multiple passes over the data, requiring untold amounts of memory or time.

Instead we introduce a staging restriction on the language, splitting into two stages:
- Element computations, based on a single stream value
- Aggregate computations, which are available after all stream elements have been read.

The restriction is that Element computations cannot refer to Aggregate computations, as Aggregate computations are only available at the end of the stream.

```
λ from injury in filter severity > mean severity  in count severity
                                 ^
Error
-----

## Check error

  Cannot discharge constraints at 1:32

    1:32  Cannot join temporalities.
          Aggregate with Element.
          Chances are the query requires multiple passes over the data.
```

Again, the last lines are the most important, as well as the position
of the error. This means that the two arguments to `>` are at different
stages, the left being Element and the right Aggregate.

Windows and dates
-----------------

It is often useful to look at just the last few months of data - this
is called windowing. Before looking at windowing proper, we should
look at the dates in the sample data. Each fact, along with its `value`
or struct fields, has a `time` attached to it. We can view the first
and last dates quite easily:

```
λ from injury ~> oldest time, newest time
C evaluation
------------

homer|(2016-01-01 00:00:00, 2016-01-05 00:00:00)
marge|(2016-01-01 00:00:00, 2016-01-05 00:00:00)
moe|(2016-01-01 00:00:00, 2016-01-02 00:00:00)
```

Now that we know the most recent entry was 2017, we can set the current snapshot date.

```
λ  :set snapshot 2016-12-31
Snapshot mode activated with a snapshot date of 2016-12-31.
```

If we wish to see the number of injuries in the year of 2016, we can write

```
λ from injury in windowed 12 months in count fields
C evaluation
------------

homer|5
marge|5
moe|2
```

Groups and distinction
----------------------

A common task is to bucket facts into different categories. Suppose we wanted to look at where the majority of injuries occur, and which tend to be the most severe. We would group by the `location` of the injury, then for each `location` find the `count` and mean with `mean severity`.

```
λ from injury in group (tolower location) in mean severity, count severity
C evaluation
------------

homer|{ "arm" -> (4.0, 3), "torso" -> (4.0, 2) }
marge|{ "head" -> (4.0, 2), "leg" -> (4.0, 3) }
moe|{ "ear" -> (1.0, 1), "hair" -> (2.0, 1) }

```


Group Folds
-----------

Finally, we permit folding over the results of a group, so if we wanted
to know which `location` has the highest rate of injury, we can do so
with

```
λ from injury in group fold (location, severity) = (group (tolower location) in mean severity) in max_by severity location
C evaluation
------------

homer|"arm"
marge|"head"
moe|"hair"
```

Custom folds
------------

Sums, counts and means are timeless reductions, but they fall short of particularly interesting analyses. It is possible to create custom traversals or folds over the stream, which is indeed how sum and count themselves are implemented. As a simple example, let us reïmplement sum as a fold.

```
λ from injury in fold my_sum = 0 then my_sum + severity in my_sum
C evaluation
------------

homer|20.0
marge|20.0
moe|3.0
```

Here we use the `let` syntax as before to give a definition a name, but instead of a simple definition, this is a fold definition. The name of the folded value is `my_sum` and its initial value is `0` - this is what the sum of an empty stream would produce. Then we have `then`.

After the initial value, each fact in the stream computes a new `my_sum`, based on the previous `my_sum` plus the current fact's `severity`.

The initial value of the fold cannot refer to any element, as it is used even if the stream is empty. There is another type of fold that only works if the stream is not empty, though, called `fold1` - fold on at least one element. With this, we can implement an exponentially rolling average:

```
λ from injury in fold1 roll = severity then severity * 0.25 + roll * 0.75 in roll
C evaluation
------------

homer|4.0625
marge|4.0625
moe|1.25
```

Here, the initial value of the fold is the value of the first element of the stream. For any subsequent facts in the stream, a quarter of the value is mixed with three quarters of the previous fold value.

Garnish with a lime twist.

