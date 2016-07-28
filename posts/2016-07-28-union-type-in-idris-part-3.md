---
title: "Union Type in Idris (Part 3)"
lang: en
category: [English, Functional Programming, Idris]
date: 2016-07-28
author: Nicolas Biri
---

Go back to the [first part](http://nicolas.biri.name/posts/2016-07-26-union-type-in-idris-part-1.html).

# Some are better than others, Union is better than sum

At this point, we have a slightly less complex syntax than sum types (with some
drawbacks, but I will detail them in another part), but did we have other
benefits?

Well, yes.

Today, we are going to inverstigate how flexible union types are. More
precisely, I will detail union type restrictions and generalisation.

# Shrink my union

Let suppose that we have a variable `x` of type
`Union [Nat, String, List String]` and the following repl session:

```
> the (Maybe Nat) x
Nothing : Maybe Nat
```

We know that `x` does not contain a `Nat`. Thus, `x` contains either a `String`
or a `List String`. Well, we can explicitly express it with union types.

Let's compute the type of this reasoning.
Given an union instance, we can either get a value of a type in this uion or we can
restrict the union. Quite clear and easy to write:

```idris
retract : Union xs -> {auto p: Elem ty xs} -> Either (Union (dropElem xs p)) ty
```

The result type may require some explanations. The presence of `Either` and the
right case (`ty`) is clear. The left case (`dropElem xs p`) is straightforward
if we look at the
[`dropElem` documentation](http://www.idris-lang.org/docs/current/base_doc/docs/Data.List.html#Data.List.dropElem): it removes the element pointed by `p` from the list.

The implementation is almost as easy to write as the type:

```idris
retract : Union xs -> {auto p: Elem ty xs} -> Either (Union (dropElem xs p)) ty
retract (MemberHere x) {p = Here} = Right x
retract (MemberHere x) {p = (There _)} = Left (MemberHere x)
retract (MemberThere x) {p = Here} = Left x
retract (MemberThere x) {p = (There later)} = either (Left . MemberThere) Right $ retract x {p = later}
```

The 3 first cases are trivial, they can almost be completed automatically by
Idris (I should talk about this really cool feature in a next post).
The last case is a bit more complex. The idea is to _lift_ the result of
retract to the next step: on a right, propagate the found value, on a left,
just buried it one step further.

It's time to see retract in action:

```
> :let x = the (Union [Nat, String, List String]) $ member "foo"
> the (Either _ Nat) $ retract x
Left (MemberHere "foo") : Either (Union [String, List String]) Nat
> the (Either _ String) $ retract x
Right "foo" : Either (Union [Nat, List String]) String
```

# Enlarge my union!

Can we do the opposite of `retract`? If I have a member of
`Union [Nat, List String]`, can we claim that we have a member of a broader
union?

Yes, we can _but_ it is a bit more complex than the `retract` case. The main
reason is that we have an infinity of target types for the result union.

## Identifying the broader unions

The objective is to define a data type that contain a proof that an union is
broader than another one. Or, by transitivity, that each element of a list
of types is contained in another list. This is as simple as it sounds. I mean,
really, if its sounds complicated to you, it will be complicated to read,
if it sounds simple, it will be simple to read:

```idris
data Sub : List a -> List a -> Type where
  SubZ : Sub [] ys
  SubK : Sub xs ys ->  Elem ty ys -> Sub (ty::xs) ys
```

To build a `Sub` type, we must be able proof that each element of the first
list are in the second one.

## Trying to `generalize` union

Now that we have a definition of `Sub`, we can type our `generalize` function
more easily:

```idris
generalize : (u: Union xs) -> {auto s: Sub xs ys} -> Union ys
```

Given an union, if we have a proof that the list of type in the union is a subset
of the list of type in the resulting union, we can generalize the union. Yes, it's
that **easy**.

Things are getting more complex when we want to implement generalize. Let's
start with a naive implementation and see how it goes:

```idris
generalize : (u: Union xs) -> {auto s: Sub xs ys} -> Union ys
generalize (MemberHere x) = member x
generalize (MemberThere x) {s = (SubK y z)} = generalize x {s=y}
```

If we have at `MemberHere`, `x` is the value of the union, and thus, must be
put in the result union, and we use previously defined `member` to do so.
If we haven't find the value yet, we just restart one step further.

This version reallly seemed ok to me. Unfortunately, it didn't match the compiler
expactations:

```
When checking right hand side of generalize with expected type
        Union ys

When checking argument p to function Data.UnionType.member:
        Can't find a value of type
                Elem ty ys
```

Almost there, the sole issue is that Idris wasn't able to compute the new
_location_ of the value. Fortunately, this location is carried along by `Sub`
and can be easily added:

```idris
generalize : (u: Union xs) -> {auto s: Sub xs ys} -> Union ys
generalize (MemberHere x) {s = (SubK _ z)} = member x {p = z}
generalize (MemberThere x) {s = (SubK y _)} = generalize x {s=y}
```

And here we go:

```
> :let x = the (Union [Nat, String]) $ member 2
> the (Union [String, Nat, List String]) $ generalize x
MemberThere (MemberHere 2) : Union [String, Nat, List String]
```

# Part 3 is over

That's all for today.
