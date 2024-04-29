---
title: Contexts
---

Icicle expressions can exist within a series of contexts which make writing
features easier. One can, for example, name sub-expressions, or
to pre-filter the data, but they can also do much more.

### Let

The simplest contexts are _let_ expressions, which allow one to calculate and
reuse values by binding them to a name.

```elm
let
  total =
    sum value

  number =
    count value
in
  total / number
```

You can bind many different expressions in a single _let_, by lining the names
up vertically or by separating them with a semicolon.

When building features however, a number of other contexts are available.

### Filter

The _filter_ context takes an expression which applies a filter to a stream
before calculating an aggregate. The value calculated within the context
must be aggregated, and the value which is being filtered must be a streaming
element


```elm
proportion : Element Bool -> Aggregate (Possibly Double)
proportion check =
  (filter check in count check) / count check

-- Or, naming the sub-expressions with a let
proportion : Element Bool -> Aggregate (Possibly Double)
proportion check =
  let
    numerator =
      filter check in count check
    denominator =
      count check
  in
    numerator / denominator
```

Returning an _element_ from a filter expression is not valid, because downstream
calculations wouldn't be able to line up between different streams if a function
were to be applied.

### Filter Let

The _filter let_ context binds a partial pattern to filter a stream before calculating
an aggregate, combining the characteristics of the preceding two contexts.

```elm
sum_options : Element (Option Int) -> Aggregate Int
sum_options optNum =
  filter let Some num = optNum
  in sum num
```

Like _filter_, returning an _element_ within the context of a _filter let_ expression
is a type error.


### Window

Window is a specialised filter which only permits facts which occurred within
a specific time window.

```elm
-- Days
windowed 14 days in sum amount
-- Weeks
windowed 2 weeks in sum amount
-- Months
windowed 1 month in sum amount
```

These are extremely useful for feature engineering tasks.

### Latest

The _latest_ contexts allows on to compute a downstream aggregation on a
limited number of values which were the most recent to arrive.

```elm
latest 10 in sum amount
```

Latests are implemented as circular buffers, holding only the required
parts of input.

### Group

The Group context allows one to create a map of results, with keys specified
by the grouping context.

```elm
group key in aggregate elements
```

within the context, an _aggregate_ must be returned, while the key must be a
streaming _element_ expression. Any calculation which produces an aggregate
is permitted inside a _group_ expression, providing a lot of flexibility.

### Fold

The _fold_ context is the key to aggregating data, and all functions producing
_aggregate_ values, such as _sum_ and _count_, are implemented using this
user facing language feature.

Folds are a programming concept similar to the _reduce_ in other languages,
which, given some initial state, (sometimes called the accumulator), will
update its state as new data points are provided. When all the data has been
seen, we can know the final state of the system.

Here's how one could defined a _sum_ expression, similar to how it is defined
in the Icicle standard library.

```elm
sum : Element Int -> Aggregate Int
sum v =
  fold acc = 0 then
    acc + v
  in
    acc
```

The critical keyword here is the context `fold`, which binds the name `acc`
as the running state. The initial value is given after the equals sign (here, 0).
Then (we use the keyword _then_ here), the state is updated for every new value
coming in.

In this example, the stream of elements is bound to the name _v_, so the new
state when calculating the sum is given by `acc + v`.

Finally, at the end of the query, we can read the value `acc`, which is an
_aggregate_ value.

As another example, here's how one can test if any values in a stream were
true.

```elm
-- Are any elements true
any : Element Bool -> Aggregate Bool
any x = fold a = False then a || x in a
```

The stream of values under test here is bound to `x`, while the accumulator
is called `a`. We start the calculation with False (as no values have been
true so far). Then, as each new data point comes in, we test if either `a`,
or the new value `x` is true. If either of them are, then our new accumulator
is going to be true as well.

Finally, we read the accumulator as the final answer, which will tell us
if at least one of the values was true.

### Scan

The _scan_ context is used to return the running values of an aggregating
query as an element.

```elm
scan running_mean = mean price
in any (price > running_mean)
```

Scanned bindings can only be used where an aggregate value is expected.

### Group and Array Fold

Group and Array fold contexts allow for the consumption of groups (maps)
and arrays as folds.

For example, the following expression will return the basket with the
highest sum of amounts.

```elm
group fold (k,v) = (group basket in sum amount)
in max_by v k
```

One can also fold over arrays using

```elm
array fold v = (...) in ...
```

The syntax is like a `let` in that the patterns on the left are bound to the
expression on the right, (which must be a `Group` for `group fold`), the
bindings are then available within the body.

Usually, when using a `group fold` it will be over the results of grouping
expression, and return an `Aggregate` value. It is however, also possible
to use a group fold to produce an element if the input `Group` is an `Element`
as well.

