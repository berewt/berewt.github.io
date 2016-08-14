---
title: "Union Type in Idris (Part 5)"
lang: en
category: [English, Functional Programming, Idris]
date: 2016-08-14
author: Nicolas Biri
---

Go back to the [first part](http://nicolas.biri.name/posts/2016-07-26-union-type-in-idris-part-1.html).

# Unit and property testing in Idris

I thought I was done with the union type series, but then I think about how
great this example is to demonstrate some of the possibilities of test and
property checking in Idris.

Idris has a [basic testing framework](http://docs.idris-lang.org/en/latest/tutorial/testing.html).
I won't discuss it here. As Idris is very young, the framework has little
interest at this time and we can do way better (in my opinion) by just
using types in a clever way.

## Unit testing at compile time

As types are a first class language construct in Idris, you can use value to
build type. An interesting type constructor is `=`. The type `a = b` is
populated by the proof that `a` is euqal to be `b`. To construct inhabitants of
this class, we often use `Refl`, which is an inhabitant of `a = a` (We already
used Refl in the definitions of isomorphisms in
[part 4](http://nicolas.biri.name/posts/2016-08-14-union-type-in-idris-part-4.html).)

This can be used to define unit testing functions, that will be resolved at compile
time. If we go back to the definition of `Union` and ot the `member` constructor

```idris
data Union : List Type -> Type where
  MemberHere : ty -> Union (ty::ts)
  MemberThere : Union ts -> Union (ty::ts)

member : ty -> {auto e: Elem ty ts} -> Union ts
member x {e = Here} = MemberHere x
member x {e = There later} =
  MemberThere (member x {e = later})
```

We can quickly set up some tests:

```idris
memberCreateMemberHereTest :
  the (Union [String, Nat]) (member "Foo") =
  MemberHere "Foo"
memberCreateMemberHereTest = Refl
```

and:

```idris
memberCreateMemberThereTest :
  the (Union [String, Nat]) (member 42) =
  MemberThere (MemberHere 42)
memberCreateMemberThereTest = Refl
```

These tests will be executed at compile time and will pass if our definition
are correct. No need for a framework.Moreover, if we ship a faulty version
without testing, thrd users won't be able to use the library because they
won't be able to compile our code.

# Theorem provind (or enhanced property testing)

Tests prove that your definitions are correct for some values, theorem prove
that it's correct for all values. Even with something as simple as the
`Union` library, we can prove fancy stuff.

## Let's `get` a `member`

First, let's recall the definition of `get`:

```idris
get : Union ts -> {auto p: Elem ty ts} -> Maybe ty
get (MemberHere x)      {p = Here}      = Just x
get (MemberHere x)      {p = There _}   = Nothing
get (MemberThere x)     {p = Here}      = Nothing
get (MemberThere later) {p = (There l)} = get later {p = l}
```

It's quite natural to see `get` as the _invert_ of `member`. Let's formalise it
with some properties.

To warm up, we'll write a first one that just work for a trivial case: one
type union. We want to check that if we build a one typ uninon with `member`
and `get` ot back, we obtain `Just`. It's as simple as writing this description:

```idris
getHereMemberIsJust : (x : a) -> get (member x) = Just x
getHereMemberIsJust _ = Refl
```
Let's go further and prove something similar for more complex unions. To do
so, we must be able to reason on _which type_ we populated. To do so, we
will make the `Elem ty ts` implicit parameter of `member` and `get` explicit.

If we create a union with `member` at a place given by `Elem ty ts` and check
the value at the same place with `get`, we should have `Just` the value:

```idris
getMemberWithElemIsJust :
  (x : a) ->
  (p : Elem a xs) ->
  the (Maybe a) (get (member x {p}) {p}) = Just x
```

This prove is trickier thant the previous one. It should be proved by induction
on `p`. The base case is easy. if `p = Here`, the compiler will reduce both part
of the equality by itself and we only has to prove a straight forward equality:

```idris
getMemberWithElemIsJust _ Here = Refl
```

The induction case is quite easy too now thanks to the previous case hypothesis.

It's time for me to introduce the interactive proof facilities of Idris. Actually,
I should have done it way earlier but it's one of this thing that seems so natural
to use that you forgot to mention how great it is.

The Idris REPL, and many compatible editors, provide some helpful command to ease
type driven development. One of them is being able to define _holes_ in your code
and to check their type. More details about TDD and interactive development in
Idris can be found in the
[online documentation](http://docs.idris-lang.org/en/latest/tutorial/interactive.html)
or in the excellent
[Idris book](https://www.manning.com/books/type-driven-development-with-idris).

Now that you know it, we can use hole in our induction case:

```idris
getMemberWithElemIsJust x (There later) = ?wut
```

`?wut` is our hole, and if we ask to the compiler its type, we obtain:

```
  a : Type
  x : a
  xs : List Type
  later : Elem a xs
  y : Type
--------------------------------------
wut : get (member x) = Just x
```

Idris provides a recap of all the types in the context, and display the
expected type for `wut`. With the context information, Idris was able,
by itself, to reduce `wut` to the type of the induction hypothesis. Thus,
we just have to provide it:

```idris
getMemberWithElemIsJust x (There later) = getMemberWithElemIsJust x later
```

And that's it. We have a complete proof that if we put a value at a given
place of a union and if we get the value at the same place, we obtain `Just`
this value.

# The difficulty of theorem proving (digression)

The above proofs are almost trivial when you are used to types and to theorem
proving. However, it may be hard to grasp for newcomers. Actually, these small
examples can be used to illustrate the difficulties encountered by the
advocates of formally proved code to convince developpers and the industry.

To be honest, I had few experiences with formal proof education to devleopers.
However, during these experiences, I encountered two majors objections:

1. The properties I want to test are not complex enough to require a whole
   proof, proving that it works for some values, or for a set of random value is
   sufficient.
2. When I have to encode a proof, I have to formalize a work that I have already
   done mentally when I wrote the code, in a non-natural way.

Both objections are unfortunately valid in most cases. We can however mitigates
this feeling by:

1. What if, for almost the same costs, as unit or properties tests, you can be
   sure that it is valid for all the values.
2. Providing helpful enough programing environment to mae the formalisation
   process as natural as possible.

Moreover, I should emphasize one of the major advantages of formal proof
embedded in a library, that I've already mentioned in the article. These
proofs of properties can be seen as a contract that my library ensure to
respect. It offers garantees about third party code and thus can higly
reduce the lack of confidence that I can have in those libraries.

