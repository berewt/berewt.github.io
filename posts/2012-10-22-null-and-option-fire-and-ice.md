---
title: Null and Option, fire and ice
comments: true
categories: English, 2 cents about Programming
tags: Option, null, Java, Scala, Groovy
---

ML, Haskell or Erlang, does not have `null` and they [don't miss it][maybe].
Tony Hoare himself consider the introduction of null as an error. A billion
dollar error more precisely. As a consequence, many languages try to work around
`null`. Groovy has its *safe navigation operator* (`.?`), Scala has its *Option*
and Java has… well Java has `if (x != null) … else …`. As this last solution is
neither concise nor elegant, some programmers support the introduction of an
equivalent to `Option` in Java. In the Java 8 perspective, [Mario Fusco's
article][mario] is certainly one of the most interesting on this subject. The
idea is simple and straightforward: as it seems cool to avoid using `null`,
let's provide a way to avoid it more often.

A few words about the Groovy solution. `x?.foo` is basically a shortcut for:
`x == null? null : x.foo` you can achieve almost the same with a macro in your
text editor. It is useful because it saves a lot of typing and makes code denser
but it doesn't provide any type safety.

We should agree that the main goal of avoiding `null` is to be sure that we
handle the lack of information properly. *Properly* means that if there is an
uncertainty, I **have to** deal with it either by propagating it (as the `.?`
groovy operator or the [null object pattern][nop] does), but also by providing a
default value or by doing a different process depending on the case.
`Option`/`Maybe` comes with a lot of handy function to do it.

So far, so good. Except that Scala and Java do have `null`. Let's come back to
what happen in a null-free world. Suppose that you're writing a function and
you're not sure you can provide the expected result. In this case, you only have
two options: 

- Use an `Option`/`Maybe` and claim explicitly that your function doesn't know
  if it can provide a result.
- Throw an exception, saying that something was not expected at all directly.

It's clear. I'm not sure it can be clearer. Let's come back to the world of
null and look how we deal with the same situation, without taking into account
the `Option` alternative:

- Return null if your function can compute a result. In the best case, you will
  say it explicitly… in the documentation. The compiler isn't aware of it.
  Other devs will know about it only if they read the doc. But the program
  continues.
- Throw an exception, exactly like in the previous case.

Adding Option to the second case, people expect to get back to the first one.
Ahem… **No!** Here is what you get instead:

- Return null if your function can compute a result. In the best case, you will
  say it explicitly… in the documentation. The compiler isn't aware of it.
  Other devs will know about it only if they read the doc. But the program
  continues.
- Use an `Option`/`Maybe` and claim explicitly that your function doesn't know
  if it can provides a result.
- Throw an exception, saying that something was not expected at all directly.

Yes, it's almost an automatic merge. Do you see where I'm going with that? When
I call a function, I can't know if the programmer hasn't decide to use `null` if
the function don't have a result. Worse, even if I see an Option, I can't tell
if this guy isn't bad enough to return a `null` instead of a real `Option`. Yes
you can expect that other programmers respect a kind of best practice and stick
to the Option/exception alternatives. But best practices aren't statically
typed.

It leads us to a difficult choice. Shall we add this third solution or not. Due
to the feedbacks I've read about Mario's article about `Option` in Java, I tend
to understand that the community thinks so. After a long discussion between
myself and I, I tend to agree with them. But I keep it mind that it doesn't
resolve at all the `null` issue. It's just a patch.

[maybe]: </English/Make%20it%20functional/2012/10/10/nothing-just-call-me-maybe/>
  "Nothing, Just, call me Maybe (on this blog)"
[mario]: <http://java.dzone.com/articles/no-more-excuses-use-null>
  "No more excuses to use null references in Java 8, on DZone"
[nop]: <http://en.wikipedia.org/wiki/Null_Object_pattern>
  "The null object pattern in Wikipedia"
