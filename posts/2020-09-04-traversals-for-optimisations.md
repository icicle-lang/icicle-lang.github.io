---
title: Traversals as Optimisations
description: How traversals over our syntax tree help make optimisation passes simple and easy to maintain
author: Huw
---

Using traversals and a simple fixpoint monad, we can efficiently crunch Icicle expressions to a minimal, efficient kernel. This speeds up downstream compilation, reduces generated code size, and makes our queries run faster.

Icicle is a high level language. The source language has mode polymorphism for error handling and runtime staging, type inference, function abstractions, and syntactic sugar to make describing features fast and easy. This is great, but due to the large surface area, makes its source language is not a great target for writing compiler optimisation passes.

Fortunately though, during compilation we pass through a simple, typed expression language called _Core_.

In this post, we'll look at how we can apply high level optimisations in _Core_ to significantly reduce our amount of code and remove redundant code branches.

Traversals
---

If we write a traversal over the _children_ of an expression tree, we can pattern match on key terms, and then trivially recurse elsewhere in the tree to find all the sub-expressions where this pattern match applies.

The `Plated` type class from the Haskell [lens] library allows us to [scrap our boilerplate] by recursively traversing syntax trees in this manner. Twan van Laarhoven, who perhaps not coïncidentally invented modern lenses, has a great [blog post][twan] about traversals like plated.

Here's the simplified definition:
```haskell
class Plated a where
  plate :: forall f. Applicative f => (a -> f a) -> a -> f a
```

And here is our core expression language[^1].

```haskell
data Exp
 -- | Read a variable from environment
 = XVar !Name
 -- | A predefined primitive
 | XPrim !Prim
 -- | A constant simple value with its type
 | XValue !ValType !BaseValue
 -- | Application
 | XApp !Exp !Exp
 -- | Lambda abstraction
 | XLam !Name !ValType !Exp
 -- | Let binding
 | XLet !Name !Exp !Exp
 deriving (Eq, Show)

data Prim
 -- | Minimal things like numeric, string, and date primitives.
 = PrimMinimal  !PrimMinimal
 -- | Fold and return type
 | PrimFold     !PrimFold !ValType
 deriving (Eq, Show)

-- | Folds for destructing things (catamorphisms)
data PrimFold
 -- | If then else fold over bools
 = PrimFoldBool
 -- | Case expression over an Option
 | PrimFoldOption !ValType
 -- | Case expression over a Sum
 | PrimFoldSum    !ValType !ValType
 deriving (Eq, Show)
```

Pretty standard stuff. Our primitive type contains a number of folds which we compile case matches to. `PrimFoldBool` for example is what an `if _ then _ else` from the source language becomes.

_It's the combination of our folds and expressions which are most interesting when it comes to optimisation rules._

Remembering that `plate` is a traversal over the _children_ of an expression tree, here is our `Plated` instance:

```haskell
instance Plated Exp where
  plate f (XApp x y) =
    XApp <$> f x <*> f y
  plate f (XLam n t x) =
    XLam n t <$> f x
  plate f (XLet n x y) =
    XLet n <$> f x <*> f y

  plate _ x@XVar {} = pure x
  plate _ x@XPrim {} = pure x
  plate _ x@XValue {} = pure x
```



This allows for extremely concise traversals, for example, we can easily count how many times a variable is used in an expression (this can be useful when figuring out if we should inline it for instance).

```haskell
foldExp :: Monoid x => (Exp -> x) -> (Exp -> x)
foldExp = foldMapOf plate

varCount :: Name -> Exp -> Sum Int
varCount i (XVar j) | i == j = Sum 1
varCount i x = foldExp (varCount i) x
```

The key transformer we'll use during optimisation however is `transformM`, which traverses every element in the tree, in a bottom-up manner.

```haskell
transformM :: (Monad m, Plated a) => (a -> m a) -> a -> m a
```

There's also a pure counterpart `transform`, but we're going to need a fresh name supply, and a way to indicate if we've reached a fix point.
We do this with a custom monad `FixT`. For this post we'll use an alternative version, as it is isomorphic to `WriterT Any` (with `Any` from `Data.Monoid`), where if we make any progress, we indicate this with a progress function instead or `pure` or `return`.

```haskell
type FixT m a = WriterT Any m a

progress :: Monad m => a -> FixT m a
progress a = do
  tell (Any True)
  return a
```
Then to reach a fixpoint, we recursively call this function until there is no more work to do

```haskell
fixpoint :: Monad m => (a -> FixT m a) -> a -> m a
fixpoint f a
 = do (a', progress) <- runWriter (f a)
      case progress of
        Any True  -> fixpoint f a'
        Any False -> return a'
```

That's all the tools we need to write our optimisation passes.


Simple Passes
---

The simplest optimisation we use is constant folding of primitive functions. If a primitive function is fully saturated with real values, we can run our core evaluator and replace the expression with the new result.

It looks something like this (where we have helper function type signatures as comments).
```haskell
-- takePrimApps :: Exp  -> Maybe (Prim, [Exp])
-- takeValue    :: Exp  -> Maybe Value
-- simpPrim     :: Prim -> [Value] -> Maybe Exp

constantFold :: Monad m => Exp -> FixT m Exp
constantFold unsimplified
  | Just (prim, args) <- takePrimApps unsimplified
  , Just valueArgs    <- traverse takeValue args
  , Just simplified   <- simpPrim prim valueArgs
  = progress simplified
  | otherwise
  = return unsimplified
```

Notice that we're matching on the whole expression here. It's extremely unlikely that the user's whole program is a single primitive application, so we need to traverse the leaves of the expression tree, finding where we might be able to constant fold. Fortunately, we have just the function:

```haskell
simpExp :: Monad m => Exp -> FixT m Exp
simpExp = transformM constantFold
```

A slightly more advanced optimisation is known as the case of known constructor optimisation. Let's have a look with an example:
```
Sum_fold#
  (\a -> False)
  (\b -> eq# b b_test)
  (right# (get_location# val))
```
Here, `Sum_fold#` is a primitive which acts like a Haskell `case` expression over an `Either`. The arguments are: the lambda of the left case; that of the right case; then the scrutinised expression.

Here, even though we don't have a `val` at compile time. We can see that the result is always going to take the right branch of the fold, and therefore rewrite this expression to
```
let b = get_location# val
 in eq# b b_test
```
The scrutinee in the example above (`right# (get_location# val)`), is always going to end up with a `Right`, even though it can't be constant folded, due to `right#` being the known constructor of a `Right` value. We refer to this as a _irrefutable_ expression.

Something More Interesting
---


A more challenging example of a pattern which appears pretty regularly due to inlining and the way in which our modal type system handles error conditions is something like this:

```
Sum_fold#
  (\a -> False)
  (\b -> eq# b b_test)
  (Sum_fold#
    (\err -> left# err)
    (\val -> right# (get_location# val))
    scrutinee)
```

The expression above is not optimal. It's not a constant though, and the scrutinee isn't guaranteed to force a single branch either, so we can't constant fold this expression or use the case of known constructor optimisation... if we look really closely though, we can see that scrutinising a second time isn't actually required at all.

We can justifiably rewrite this expression as:

```
Sum_fold#
  (\err ->
    let a = err
     in False)
  (\val ->
    let b = get_location# val
     in eq# b b_test)
  scrutinee
```


The key insight here is that if we scrutinise the `scrutinee`, and find a `Right` value, we're always going to exercise the `Right` case of the outer fold; and similarly for a `Left` case we know what side we'll receive. _Both_ sides of the expression are _irrefutably_ going to be a `Right` value or a `Left` value, even though we don't know what values they will hold.

We can therefore skip the outer case expression, in what we call the _Case of Irrefutable Case_ optimisation[^2].

Restating what we saw above: if we _know_ which cases both branches of the inner expression will produce, we don't actually need to check them again.

This function as implemented in our compiler is actually a little hairy, as it also sees through let bindings, handles other types of folds, and takes care of shadowing and renaming, so I won't reproduce it here. Crucially though, it does not worry about recursion and tree traversal and reports progress to our fixpoint monad. We can therefore compose it with our constant fold and case of known constructor passes.


```haskell
simpExp :: Monad m => Exp -> FixT m Exp
simpExp =
  transformM transformations
    where
  transformations
    =   constantFoldExp
    >=> caseOfKnownConstructor
    >=> caseOfIrrefutableCase
    >=> inline
```

_This insight is what makes our optimisations efficient._ We have a little bundle of optimisation functions which act in this manner, all apply to a single expression in the tree. By combining them here, at the single leaf or node, we can crunch individual leaves to their optimal state, before repeating the process for their parents. We can then do this process just a few times to reach a fixpoint. If we were to run each optimisation function independently leaf to root, we'd need to traverse the whole expression a lot more.

Our whole _Core_ optimisation pipeline, is essentially this:

```haskell
crunch :: Exp -> Fresh Exp
crunch = fixpoint simpExp
```

And using it, we reduced code sizes by up to 70%, with a commensurate reduction in downstream compilation time and a healthy boost to our runtime performance.


 [lens]: https://hackage.haskell.org/package/lens
 [twan]: https://twanvl.nl/blog/haskell/traversing-syntax-trees
 [scrap our boilerplate]: http://community.haskell.org/~ndm/uniplate/


[^1]: This is actually slightly simplified in that I've specialised or removed unnecessary type parameters. All of the case branches of `Exp` are present, while some folds and primitives are elided.


[^2]: GHC, being a highly optimising compiler, has a similar optimisation called the _Case of Case_ optimisation. The difference is that in GHC, the outer case expression is duplicated in its entirety. In the cases where we would do _our_ optimisation, a case of known constructor would apply and reach the same result. Practically speaking, in GHC, some heuristics optimistically decide when to perform a _Case of Case_; while in Icicle, a _Case of Irrefutable Case_ is always a good idea.
