---
title: "Union Type in Idris (Part 4)"
lang: en
category: [English, Functional Programming, Idris]
date: 2016-08-02
author: Nicolas Biri
---

Go back to the [first part](http://nicolas.biri.name/posts/2016-07-26-union-type-in-idris-part-1.html).

# A glimpe of equality

Idris handles equality with a typeclass, just like Haskell. Whilst equality
must be define specifically for each sum type, we can define the equality in a
generic manner for union types. The idea is to define the equality iteratively
on the list of types of the union.

## Equality for `Union x::xs`

Let's start with the recursive case, do we really need to elaborate?

```idris
(Eq ty, Eq (Union xs)) => Eq (Union (ty::xs)) where
  (==) (MemberHere x) (MemberHere y) = x == y
  (==) (MemberHere x) (MemberThere y) = False
  (==) (MemberThere x) (MemberHere y) = False
  (==) (MemberThere x) (MemberThere y) = x == y
```

We need equality for the previous union (induction) nd for the newly added type.
Then, two terms are equal if they point to the same type and have the same value.

## The base case

Things here are a little trickier. We nned an instance of `Eq (Union [])`.
Let's look back at the `Union` definition:

```idris
data Union : List Type -> Type where
  MemberHere : (x: ty) -> Union (ty::xs)
  MemberThere : (x: Union xs) -> Union (ty::xs)
```

Do you see the potential issue? None of them allow us to build a `Union []`.
It's impossible to build an element of `Union []`? Just explain it to Idris:

```idris
Eq (Union []) where
  (==) (MemberHere _) _ impossible
  (/=) (MemberThere _) _ impossible
```

Profit:

```idris
> the (Union [Nat, String]) "foo" = the (Union [Nat, String]) "foo"
True : Boolean
> the (Union [Nat, String]) "foo" = the (Union [Nat, String]) "bar"
False : Boolean
> the (Union [Nat, String]) "foo" = the (Union [Nat, String]) 42
False : Boolean
```

## Uninhabited types

There is an alternative to the `impossible` solution. We can claim that a type
is uninhabited thanks to a typeclass:

```idris
Uninhabited (Union []) where
    uninhabited (MemberHere _) impossible
    uninhabited (MemberThere _) impossible
```
`Uninhabited` comes with an `absurd` funciton of type `Uninhabited e => e -> a`
absurd can be read as: if we have an instance of an uninhabited type, we can
build anything from it.

Thanks to `Uninhabited`, our instance of `Eq` can be written:

```idris
Eq (Union []) where
  (==) x _ = absurd x
```

# Unions and synonyms

If our unions are well defined, a union of a unique type should be isomorphic
to this type. A union of two types should be isomorphic to `Either`. Let's
transform these assumptions into proofs.

## Morphisms

The morphisms can be implemented as instances of the `Cast` typeclass and are
straightforward as soon as we know how to deal with uninhabited types:

```idris
Cast (Union [ty]) ty where
  cast (MemberHere x) = x
  cast (MemberThere x) = absurd x

Cast l (Union [l]) where
  cast x = (MemberHere x)

Cast (Union [l, r]) (Either l r) where
  cast (MemberHere x) = Left x
  cast (MemberThere (MemberHere x)) = Right x
  cast (MemberThere (MemberThere x)) = absurd x

Cast (Either l r) (Union [l, r]) where
  cast (Left x) = (MemberHere x)
  cast (Right x) = (MemberThere (MemberHere x))

```

## Put an iso in our morphisms

Idris has a `Iso` type, that is use to prove that two types are isomorphic.
Here is its definition:

```idris
data Iso : Type -> Type -> Type
  MkIso : (to : a -> b)
        -> (from : b -> a)
        -> (toFrom : (y : b) -> to (from y) = y)
        -> (fromTo : (x : a) -> from (to x) = x)
        -> Iso a b
```

`to` and `from` are the two morphisms used to build the isomorphism. `toFrom`
and `fromTo` are proofs. They are populated if and only if for (`to . from`)
and (`from . to`) behave like the `id` function.

Thus, being able to build an instance of `Iso a b` is a proof that `a` and `b`
are isomorphic. Let's buils these instances for `Union`:

```idris
oneTypeUnion : Iso (Union [ty]) ty
oneTypeUnion = MkIso cast cast toFrom fromTo
  where
    toFrom _ = Refl
    fromTo (MemberHere _) = Refl
    fromTo (MemberThere x) = absurd x
```

And:

```idris
eitherUnion : Iso (Union [l, r]) (Either l r)
eitherUnion = MkIso cast cast toFrom fromTo
  where
    toFrom (Left _) = Refl
    toFrom (Right _) = Refl
    fromTo (MemberHere _) = Refl
    fromTo (MemberThere (MemberHere _)) = Refl
    fromTo (MemberThere (MemberThere x)) = absurd x
```

`Refl` is a member of the type `x = x`.
Being able to compile this code build a formal proof of the existence of the isomorphisms.

# Part 4 is over

That's all for this serie, the code is [on github](https://github.com/berewt/UnionType).
