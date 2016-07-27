---
title: "Union Type in Idris (Part 1)"
lang: en
category: [English, Functional Programming, Idris]
date: 2016-07-26
author: Nicolas Biri
---

**TL;DR:** This article discusses the interest of union types and presents an
implementation of this concept in Idris.

# An introduction to sum and union types

If you are familiar with language like Haskell or Scala, you may know what a
sum type is. Consider the following example in Haskell:

```haskell
data Whisky = Whisky {age :: Nat, alcohol :: Float}
data Beer = Beer {alcohol :: Float}

data Alcohol = AlcoholWhisky Whisky | AlcoholBeer Beer

myAlcohol :: Alcohol
myAlcohol = AlcoholWhisky (Whisky 12 40)
```

In this short example, `Alcohol` is a sum type. It's called like this because
the number of inhabitants of this type is the sum of the inhabitants of the type
`Whisky` and of those of the type `Beer`.

I always thought that the need of the `Alcohol` constructors (`AlcoholWhisky`
and `AlcoholBeer`) is a little clumpsy in such types. It's getting worse when we
stack sums:

```haskell
data Water = Water {calcium :: Float}
data OrangeJuice = Water {sugar :: Float}

data NoAlcohol = NoAlcoholWater Water | NoAlcoholOrangeJuice OrangeJuice

data Beverage = BeverageAlcohol Alcohol | BeverageNoAlcohol NoAlcohol

myBeverage :: Beverage
myBeverage = BeverageAlcohol myAlcohol
```

It would be easier if we can claim that a type `t` is a union of several types and
provides any of these types vaule as a value of type `t`.

This idea is not new, it's for example investigated in the
[`open-union`](http://hackage.haskell.org/package/open-union) package. Thanks to
this package, we can replace the first example with something like:

```haskell
type Alcohol = Union '[Whisky, Beer]


myAlcohol :: Alcohol
myAlcohol = liftUnion (Whisky 12 40)
```

`open-union` is a great solution in Haskell. However, it relies on
`Data.Dynamic` which means that we need to carry over the representation of the
type.

# Defining union types in Idris

[Idris](http://www.idris-lang.org/) is a purely functional programming language
with dependent-type designed by Edwin Brady. I assume here a basic knowledge of
Idris and of dependent types.

## Declaring union types

Suppose that you can promote the values as types, how would it helps to build
an union type? This question was the one that started the journey.

Here was the idea: we can define a type as a list of type. From there, it
suffices to find a way to "point" the valid type in our union and we obtain an
union for free. Here is the implementation:

```idris
data Union : List Type -> Type where
  MemberHere : ty -> Union (ty::ts)
  MemberThere : Union ts -> Union (ty::ts)
```

So, `Union` is a type that is parametrized by a list of types. For example, with
this data declaration, `Union [Char, String]` is a valid type.

To build members of `Union` we have two constructors. `MemberHere` is the
easiest. Given a value of type `ty`, we can build an instance of any `Union`
type such that the list of its composed types starts with `ty`, whatever are the
other types of the union. `MemberThere` provides a way to prepend other types in
our `Union`. Let's see these types in action:

```idris
x : Union [String, Nat, List String]
x = MemberHere "Ahoy!"

y : Union [String, Nat, List String]
x = MemberThere (MemberHere 3)
```

With this basic definitions, we have in hands all we need to create union types.
Altough, the instances are really painful to write, even more than with classic
sum types.

We can do way better with a small helper:

```idris
member : ty -> {auto e: Elem ty ts} -> Union ts
member x {e=Here} = MemberHere x
member x {e=There later} = MemberThere (member x {e=later})
```

This short function takes advantages of Idris
[auto implicit arguments](http://docs.idris-lang.org/en/latest/tutorial/miscellany.html#auto-implicit-arguments),
which automatically computes argument at compile times, depending on the
executino context. The idea is that, given a type `ty`, such that `ty` is in the
union, we can compute the location of this type in the union and provide an
`Elem`. This `Elem` is then used to compute the `Union` boilerplate. An we obtain:

```idris
x : Union [String, Nat, List String]
x = member "Ahoy!"

y : Union [String, Nat, List String]
x = member 3
```

Note that as complex as the union will be, the instanciation will always
straightforward.

## Extracting union type

Ok, so union type are easy to declare and to instanciate. We also need an easy
way to get our value back if we want to compete with sun types.

Let's define a `get` function that extract a value of type `ty` from an union.
To typecheck, `ty` must be a valid type (a type listed in the union). And we can't
be sure to obtain a `ty`. Thus, the type of `get` is:

```idris
get : Union ts -> {auto p: Elem ty ts} -> Maybe ty
```
And once again, the idea is to use the witness of the type position in the
union (`p`) to compute the answer.

```idris
get : Union ts -> {auto e: Elem ty ts} -> Maybe ty
get (MemberHere x)       {e=Here}    = Just x
get (MemberHere x)       {e=There _} = Nothing
get (MemberThere x)      {e=Here}    = Nothing
get (MemberThere later) {e=(There l)} = get later {e=l}
```

All these case should be straight forward. And now we can have (in the REPL):

```
>>> :let x = the (Union [String, Nat, List String]) $ member "Ahoy!"
>>> the (Maybe String) $ get x
Just "Ahoy!" : Maybe String
>>> the (Maybe Nat) $ get x
Nothing : Maybe Nat
>>> 
```

We uset `the` here to provide a Type hint, but it won't be useful in most of
the cases, as the type will be provided by the context.

# Part 1 is over

That's all for today, you can continue with the fold for
[union type](http://nicolas.biri.name/posts/2016-07-27-union-type-in-idris-part-2.html).


