---
title: Records
---

Records are a light-weight type with named fields which are extremely easy to access
in a clear way.

In Icicle, records will often be used as the input for queries, but they can also be
inputs and returned values of functions; and used to make queries easier to understand.

Records have no runtime cost, and are flattened to their individual fields in generated
C code.

Comparisons to other languages
------------------------------

Records in Icicle are very similar to that of Elm or Purescript, and quite similar to
objects in JavaScript.

When comparing records to JavaScript:

- It is a type error to attempt to extract a field from a record which does not exist;
- Fields in a record will always exist and be of the correct type; and
- There is no 'this' keyword or internal hidden state.


Creating and Accessing Records
------------------------------

Records are very easy to create syntactically, in a way which will seem familiar to
JavaScript developers and others.

For example, a record from a library might reference books:
```elm
{ author = "Becky Chambers",
  title = "A Closed and Common Orbit",
  available = False,
  acquired = 2020-10-05,
}
```

The type will be inferred as:
```elm
{ acquired : Time,
  author : String,
  available : Bool,
  title : String
}
```

To retrieve a value from a record, one can use a 'dot' syntax.

```elm
let
  book =
    { author = "Becky Chambers",
      title = "A Closed and Common Orbit",
      available = False,
      acquired = 2020-10-05,
    }
in
  book.author
```

Some other values can use record dot syntax as well, namely,
times. Although times are not implemented as records, one can
extract the year, month, and day from as such

```elm
let
  book =
    { author = "Becky Chambers",
      title = "A Closed and Common Orbit",
      available = False,
      acquired = 2020-10-05,
    }
in
  book.acquired.year
```
