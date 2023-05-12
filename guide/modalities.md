---
title: Modalities
---


Icicle's has a principled type system, meaning that every expression in
the language is both able to be inferred, and has a single most general
type.

What this means in practice is that no expressions need type annotations,
and if queries make logical sense, they'll type check without any extra
work from the programmer.


One critical component of Icicle is its Modal type system, which is
used to ensure streaming runtime characteristics for queries as well
as track error conditions.


Modal type system stem from Modal Logic[^wiki] and allow Icicle expressions
to allow to have a simple syntax while still ensuring the streaming
properties of queries.


Modal types qualify proof terms, so we can propose that the result of a
division is `Possibly Double`. Icicle uses modalities for error conditions,
as well as, when in the computation a variable is available.
Elements of the input are tracked as `Element`, while at the end, all
values must be `Aggregate` or `Pure`.


Application through modalities is implicit, so operators like `+` will work for
all modes, but not necessarily their combinations; as adding an `Element`
to an `Aggregate` is forbidden.

The key type error one will see with regards to modalities is this:

```
Cannot join temporalities.
Element with Aggregate.
Chances are the query requires multiple passes over the data.
```

Which means that the modal type system has been violated, and the query
doesn't make sense to perform in a single pass over the data.


[^wiki]: [Wikipedia Modal_Logic](https://en.wikipedia.org/wiki/Modal_Logic)
