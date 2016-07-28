---
title: "Union Type in Idris (Part 2)"
lang: en
category: [English, Functional Programming, Idris]
date: 2016-07-27
author: Nicolas Biri
---

The [first part](http://nicolas.biri.name/posts/2016-07-26-union-type-in-idris-part-1.html).

# First part summary

We have detailed how we can write union types in Idris and how to extract a
value from it. We recall here the union type definition, that is used through
this article:

```idris
data Union : List Type -> Type where
  MemberHere : ty -> Union (ty::ts)
  MemberThere : Union ts -> Union (ty::ts)
```

You may also need the signature of the previously defined function for creating
union type values and retrieving the value of an union:


```idris
member : ty -> {auto e: Elem ty ts} -> Union ts
get : Union ts -> {auto p: Elem ty ts} -> Maybe ty
```

# Folding Union

When I started reading some Idris, I was amazed by the
[`printf` example](https://gist.github.com/puffnfresh/11202637)
(also available [in video](https://www.youtube.com/watch?v=fVBck2Zngjo)).

And my motivation when I start union type was actually to provide a type safe
way to fold an union type.

## A first try: Decomposing the union type

So my first idea was to decompose the union to know which functions we need
to cover all the cases.. It means that I wanted something like:

```idris
> :t foldUnion (the (Union [Char, String]) $ member 'c')
foldUnion (the (Union [Char, String]) $ member 'c') : (Char -> a) -> (String -> a) -> a
```

if you want to do such a thing, it means that the type of `foldUnion u` where
`u` is an arbitrary union will _depend_ on the type of `u`. So it means that
our code will be like:

```idris
foldUnion : (u: Union ts) -> UnionFold ts
```

Where `UnionFold` should build a type from a list of type.

```idris
UnionFold : List Type -> Type
```

The case decoposition is quite straightforward atually. If the list is empty,
we are done, so the type should be an arbitrary `a`. If there is a type in
the list, we should provide a function for this type and continue with the tail
of the list:

```idris
UnionFold : List Type -> Type
UnionFold [] = a
UnionFold (ty:ts) = (ty -> a) -> UnionFold ts
```

Let's come back to `foldUnion`. The next issue we face is once again is to
choose the right function to apply:

```idris
foldUnion : Union ts -> UnionCata a ts
foldUnion (MemberHere x) = \f => foldUnion' (f x)
  where
    foldUnion' : {xs : List Type} -> (x : a) -> UnionCata a xs
    foldUnion' {xs = []} x = x
    foldUnion' {xs = _::xs'} x = const $ foldUnion' {xs=xs'} x
foldUnion (MemberThere later) = const $ foldUnion later
```

If we have the `MemberHere` constructor, it means thas the element of the
union has the first type of the union, and thus, we need to apply the first
function. The challenge is then to skip all the other functions until we
exhaust all the mapping functions.

If we are in the `MemberThere` case, we must skip the next function and look
further.

We can know write things like:

```idris
>>> :let x = the (Union [String, Nat, List String]) $ member "Ahoy!"
>>> foldUnion x length id (sum . map length)
5 : Nat
```

## Functions to the left, union to the right

There is an issue with the former proposal: we need to provide the unon first.
It makes sense in the case of `printf`, as we want to provide the format string
before its parameter. It's less meaningful in the case of folding.

It's pretty hard to change the parameter order though. Actually, if we dont know the
union, how can we know if we still have to pass a function or the final union.
More clearly, we aren't able to define the type of a partially applied function.

An alternative is to gather the folding functions to define what kind of unions can
be fold with these functions.

The objective is consequently to obtain a signature like:

```idris
foldUnion : UnionFold a ts -> (u: Union ts) -> a
```

With this signature, `UnionFold a ts` indicates that we have sufficient
functions to fold an `Union ts`, to provide an `a`. My goal was to be able to
build this UnionFold as easily as possible. I take advantage of the
[list syntax in Idris](http://docs.idris-lang.org/en/latest/tutorial/typesfuns.html#list-and-vect)
to obtain a convenient way to define it. So `UnionFold a ts` is defined as:

```idris
data UnionFold : (target: Type) -> (union: Type) -> Type where
  Nil : UnionFold a (Union [])
  (::) : (t -> a) -> UnionFold a (Union ts)
                  -> UnionFold a (Union (t::ts))
```

This definition is straightforward as soon as we have understand the objective
of this type.

With this, we can know detail `foldUnion`:

```idris
foldUnion : (fs: UnionFold a (Union ts)) -> Union ts
                                         -> a
foldUnion [] (MemberHere _) impossible
foldUnion [] (MemberThere _) impossible
foldUnion (f :: _) (MemberHere y) = f y
foldUnion (f :: xs) (MemberThere y) = foldUnion xs y
```

The two first cases (ithe impossible ones) can be omited (as they are
impossible). I added them for documentation purpose. The two last cases
are quite easy to read: we go through the `UnionFold` and the `Union`
simultaneously and as soon as we fan the right location, we apply the
corresponding function.

And the previous example became:

```idris
>>> :let x = the (Union [String, Nat, List String]) $ member "Ahoy!"
>>> foldUnion [length, id, sum . map length] x
5 : Nat
```

# Part 2 is over

That's all for today, next time I will talk about union
[restriction and generalization](http://nicolas.biri.name/posts/2016-07-28-union-type-in-idris-part-3.html).



