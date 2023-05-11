---
title: Syntax
---

Icicle has a small syntax, designed to look like simple functional
languages. It's whitespace sensitive (a bit like python) and is
designed to be concise and clear.

Comments
--------

```elm
-- a single line comment

{- a multiline comment
   {- can be nested -}
-}

True -- It's true, comments can come after expressions
1 +  {- and in the middle of them -} 2
```

Literals
--------

Icicle support a wide range of literals, including dates.

```elm
()         : Unit

True       : Bool
False      : Bool

1          : Num a => a -- Can be either an Int or Double
1.4        : Double

"one"      : String
2020-01-01 : Time
```

Function Application
--------------------

Applying functions has a very minimal syntax, just whitespace
is used to separate a function from their arguments.

```elm
-- apply the sum function to the argument amount
sum amount
```

To apply to multiple arguments, just use another space
```elm
-- max_by takes two arguments
max_by total_sales store
```



Branching on Conditionals
-------------------------

The simplest conditional is an if expression
```elm
if value > 10 then
  Some value
else
  None
```

You can also pattern match on expressions
```elm
case total of
  Some value then
    value
  None then
    0
```

Let Expressions
---------------

Let expressions provide a simple context under which a
new name is available.

```elm
let
  total =
    sum value

  number =
    count value

in
  total / number
```

One doesn't usually need to put semi-colons between expressions
in a let statement, just line them up vertically. But, if it's
nicer to put them on a single line one can explicitly separate
them

```elm
let a = 2; b = 3 in a + b
```

Let expressions are the simplest contexts, but others are critical
too, including filters, folds, windows, and group bys.



Function Definitions
--------------------

Users can trivially write helper functions and custom folds to
make writing queries easier.

```elm
mean a = sum a / count a
```

The arguments which must be provided to `mean` here are specified
before the equals sign (i.e., a), and are available in the body
of the function.


Type Signatures
---------------

Type signatures are never required when writing features, but can
be used to document and check functions. Type signatures come after
a single colon.

```elm
three : Int
three = 3
```

Function types include arrows between their arguments to their result.

```elm
add : Int -> Int -> Int
add a b = a + b
```

Types which start with a capital letter are fixed types, while those
with a lower case letter are polymorphic types, meaning the caller
of the function can decide what type to use.

```elm
identity : a -> a
identity a = a
```

The function identity simply returns the value passed to it, and can
be passed a value of any type.


Some types may also contain constraints on the types which can inhabit
them

```elm
four : Num a => a
four = 4
```

The above can be read as

> for all types 'a', given that 'a' is a number; four is a value of type 'a'


Input Definitions
-----------------

Streams of data are required to have their type defined. Any type of data
is available, including complex records. When a record is used as an input,
the fields of the record are bound to names automatically.

```elm
input age : Int
```

Building Queries
----------------

Queries start with their name, and the data source which will be queried

```elm
feature total_sales =
  from sales
    in sum amount
```

Modules
-------

Modules are used to separate files and allow collaboration. Modules can
contain imports (which must come first), input declarations, function
definitions, type definitions, and features.


```elm
module Age where

import Helpers

type Adult
  = Adult ()
  | Child ()

input age : Int

adult : Int -> Adult
adult i =
  if i >= 18 then
    Adult ()
  else
    Child ()

feature is_adult =
  from age
    in adult (newest value)
```
