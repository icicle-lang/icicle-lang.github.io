---
title: >
  Receiving Gifts – a Reflection Tutorial
description: A Short Introduction to, and Explanation of, the Reflection Library
author: Huw Campbell
---

Haskell's reflection library is a powerful tool to have in one's belt, allowing the
creation of _ad hoc_ type classes informed at runtime; and as neat way of providing
runtime configuration without the need for a monad stack.


Internally though, it can be quite challenging to understand, as it can't be represented
in Haskell 98, and relies on knowledge of how GHC compiles Haskell type classes.


Motivations
-----------

A number of Haskell libraries use type classes as part of their API. Aeson for example
has `FromJSON`, which describes how to parse json values into a particular type; and
the avro library extensively uses `FromAvro`.

Normally, this offers quite good ergonomics, but it can start to be problematic when
parsing richer types like an Icicle `Value` (a value in our language).
The issue is that the json or avro schemas for a `Value` may depend on its type in the
Icicle type system, so we can't infer all we need to know to parse a `Value` from
its type alone.

<!--
Here's a simplified snippet which we use to load Avro values with a given schema, the
functions provided by the Schema registry library are based on a FromAvro instance,
and it isn't ergonomically viable to pass simple functions to them.


```haskell
-- | Wrapper which holds the function to read a value into our type.
newtype FromAvroDict a =
  FromAvroDict {
    fromAvro' :: Avro.Value -> Either String a
  }

-- | Use a given instance to instantiate a FromAvro instance.
instance Given (FromAvroDict a) => Avro.FromAvro a where
  fromAvro = fromAvro' given

-- | Given an Icicle Schema, give an action which will parse
--   bytestring encoded with a Confluent compatible encoder;
--   and deconflicting using the schema registry.
schemaRegistryDecoder
  :: MonadIO m
  => SchemaRegistry
  -> Avro.TypeName
  -> Icicle.Schema
  -> Either SchemaEncodeError
            (ByteString -> m (Either String Icicle.Value))
schemaRegistryDecoder registry tn schema = do
  avroSchema <-
    encodeColumnSchema tn schema

  -- The decodeWithSchema function uses a FromAvro instance
  -- to know the reader schema for gathering information from
  -- the Confluent schema registry. Here, we use give to
  -- supply one.
  return $ \payload ->
    give (FromAvroDict (decodeValue schema)) $
      decodeWithSchema registry avroSchema payload
```
 -->



As a separate motivation, we can solve "the configuration problem", to quote [Joachim Breitner],
this is

> finding a convenient way to work with values that are initialized once and used in many
> places all over the code.

We'll start with the `Given` class, which is the little brother of `Reified`. It's not nearly
as safe, but it's a good starting point for pedagogy.


Given
-----

For the `Given` API, there are only two items exposed. There is the `Given` type class:

```haskell
class Given a where
  given :: a
```

and the API is complete with the function
```haskell
give :: forall a r. a -> (Given a => r) -> r
````


The `Given` type class can obviously just give a value of type `a` given an instance for the
type, but here's the thing: there _are no instances_ for `Given`.


Even though there are no instances for `Given`, the `give` function "magics" one up for us, which is
then available for the inner continuation. This allows us to do really cool things!


Firstly, we can solve our issue from above. If we can `give` a function with the type of `fromAvro`
for our value type, and now  use any API which requires a `FromAvro` instance.

```haskell
instance
  Given (Avro.Value -> Either String Icicle.Value) =>
    Avro.FromAvro Icicle.Value
  where
    fromAvro = given
```

And use functions which require a `FromAvro` instance.


For the configuration problem: using a given, we can, for example, read environment and command
line parameters, then call `give` with our parsed configuration. All that is required is we
update type signatures on our internal functions with a `Given Configuration` constraint.



How Does it Work?
-----------------

Haskell as defined can't actually do _any_ of this. All class instances must be specified at compile time
and be coherent. So how does it actually work? To understand, we'll need to know how newtypes and type
classes are represented once we reach Haskell's core language.


In GHC, there are two forms of data declarations, `data` and `newtype`. Data declarations compile to
something like structures, with pointers to haskell thunks; while newtypes don't have any runtime overhead
_at all_, they just compile to the wrapped data type.


GHC, when faced with a type class constraint, will rewrite the function to take a dictionary function
argument. These dictionary arguments look a lot like data declarations, so a function like this:

```haskell
max :: Ord a => [a] -> a
max as = head (sort as)
```

Will turn the one lambda into 3, one for the type variable `a`, one for the Ord dictionary for `a` and
one for the actual arguments `as`. Here it is will a little bit of renaming:

```
max :: forall c. Ord c => [c] -> c
max = \(@ a_type) ($dOrd_A :: Ord a_type) (as :: [a_type]) ->
      head @ a_type (sort @ a_type $dOrd_A as)
```

Squinting, one should be able to see that the first lambda is the type for `a`; the next, named `$dOrd_A`
is the dictionary for `Ord a`; and finally, we have the actual list of values. The other thing to note is
that the arguments get passed along. So `head` receives 2 values, the type and the sorted list; while `sort`
takes 3, including the type, dictionary, and original values.

An interesting thing happens though with a type class like `Given`. Here, there is only a single term
`give` in the typeclass. This means that it gets treated much more like a newtype than a data constructor,
and therefore, instead of being wrapped behind a structure and thunk, it gets passed directly.

As an example, here's a simple function which adds 3 to a given integer:

```haskell
adder :: Given Int => Int
adder = given + 3
```

Looking at the core, we can see something interesting:

```
adder :: Given Int => Int
adder = \($dGiven :: Given Int) ->
  (+)
    @ Int
    GHC.Num.$fNumInt
    ($dGiven `cast` (Givens.N:Given[0] <Int>_N :: Given Int ~R# Int))
    (GHC.Types.I# 3#)
```

the addition function is supplied 4 arguments, the type `@ Int`, as well as the dictionary for
`Num Int`. The addition function is then applied to two more arguments:
  - the passed dictionary (after a cast)
  - and 3.

That meaning, there's no data constructor or extra boxing wrapping the `Given` value – and it
looks more like a `newtype` to the compiler.

This means that a function like
```haskell
adder :: Given Int => Int
```
once it hits core, looks _exactly_ like a function
```haskell
add :: Int -> Int
```

This is the critical optimisation which allows one to implement the `give` function. A newtype wrapper called
`Gift` is used to hold on to the type class constraint, so its position in the generated Core can't
float to the outside of the give function.

```haskell
newtype Gift a r = Gift (Given a => r)
```

This lets us write the `give` function, by acknowledging that this newtype has the same shape
as the function `a -> r` and using an `unsafeCoerce` to get GHC to accept this:

```haskell
give :: forall a r. a -> (Given a => r) -> r
give a k = unsafeCoerce (Gift k :: Gift a r) a
```

Now of course, `unsafeCoerce` is super dangerous and up to the mercy of the compiler, and indeed,
whether this function is inlined has caused issues in the past. But the other big issue, which makes
`Given` not ideal for most situations, is that there's no reasonable way to say what will happen
if two `give` calls are nested for the same type.


Reified
=======

The solution to this issue, is that instead of just one `Given` constraint, we use an infinite number of
existentially quantified `Reified s` constraints. Where for each call to  `reify`, we just make up a new type.

The API looks like this:
```haskell
class Reifies s a | s -> a where
  reflect :: proxy s -> a

reify
  :: forall a r. a
  -> (forall s. Reifies s a => Proxy s -> r)
  -> r
````

Now this does look a bit funnier. The key is that within the function body passed to `reify`, the `s` type is
existentially quantified, and the `s` types of two or more invocations can't unify. If that sounds familiar,
it might be because the same trick is used to isolate mutable state within the `ST` monad.

To allow one to actually call `reflect` with the phantom type – a `Proxy s` value is provided.

The implementation of `reify` is unsurprisingly very close to that of `give`, except the newtype wrapper is
called `Magic`, and we have to also pass the `Proxy` arguments.

Here it is in full:
```haskell
newtype Magic a r = Magic (forall s. Reifies s a => Proxy s -> r)

-- | Reify a value at the type level, to be recovered with 'reflect'.
reify
  :: forall a r. a
  -> (forall s. Reifies s a => Proxy s -> r)
  -> r
reify a k =
  unsafeCoerce (Magic k :: Magic a r) (const a) Proxy
```

The key is that the typeclass `Reifies` also has only one member, so acts just like its single function in core.
The supplied value `a` is used as `const a` to ignore the extra `Proxy` argument passed.


History
=======

That's kind of the core of the current library, with the technique of exploiting the semantically identical Core
representations from Ed Kmett.

Originally the implementation from Oleg Kiselyov and Chung-chieh Shan was less efficient, but more critically
couldn't easily reflect functions, as the reified values had to be an instance of `Storable`. It's an interesting
paper to read, but isn't required for understanding and using the library today.


[Joachim Breitner]: https://www.joachim-breitner.de/blog/443-A_Solution_to_the_Configuration_Problem_in_Haskell
[Austin Seipp]: https://www.schoolofhaskell.com/user/thoughtpolice/using-reflection
