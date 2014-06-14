---
title: Nothing, Just, Call me Maybe!
comments: true
categories: English, Make it functional
tags: Haskell, Java, Maybe, monad, null
---

Today, we will talk about null pointer exception. Or more precisely the lack of
null pointer exception in functional programming. Please hold 
[your trolls][beust], we are analyzing facts. It's not that functional
programming doesn't like `null`, it's just that we actually don't have this
concept. Yes we can live without it, wanna see how?

But what happens when a Haskell programmer wants to express that a function has
no result? Well, it's quite simple, we use `Maybe`. It's a monad but I'm not
going to bother you with the monad stuff at the moment, a lot was already
written on this topic, in many cases by very clever people. The Internet
doesn't need my 2 cents on it. More than that, telling you what a monad is may
be insufficient to help you getting why it's useful, especially at this stage.

So, we have the `Maybe` type, which has two constructors `Nothing` and `Just`
with the following definition:

~~~~ {.haskell}
data Maybe a = Just a
              | Nothing
~~~~

The `a` here is the way we introduces polymorphism in Haskell. Shortly, we can
either have a value boxed of type `a` in a `Just` object or `Nothing`. Imagine
you boxed your result type in Java: if it's a null, you return an object of
class `Nothing`; if it's a non null result, boxed it in a `Just` object. If you
can imagine it, you get the whole idea.

At this stage, you may wonder how it can help. Here we are:

## A bit of genealogy

We're completing our `Person` class, adding a `father` field to it.
Unfortunately, all of our people aren't really into our big brother approach,
and some of them refuse to fill the information.

Suppose that given a person, we want to produce the list of its known ancestors'
name (the list order is up to you).  How would you do that? Of course, this
example is totally stupid, but it ilustrates a classical scenario of navigation
between 0-1 cardinality properties, which happens quite frenquently in the real
world. Here is how we represent a person in Java:

~~~~ {.java}
public class Person {

    private final String firstname;
    private final String lastname;
    private final int    age;
    private final Person father;

    public Person(String myFirstname,
                  String myLastname, 
                  int myAge,
                  Person myFather) {
        this.firstname = myFirstname;
        this.lastname  = myLastname;
        this.age       = myAge;
        this.father    = myFather;
    }

    /* I still won't write getters and equals/hashCode.
     * And I still thinks it's stupid to have to.
     */
}
~~~~

And in Haskell:

~~~~ {.haskell}
data Person = Person { firstname :: String
                     , lastname  :: String
                     , age       :: Int
                     , father    :: Maybe Person
                     }
~~~~

Ok, the type signature of `father` in haskell is a bit more complex than the
java one, I admit. Let's look to the solution to our example.

## The billion dollars mistake

Let's start ith Java. We build a `Genealogy` class with an ancestor method, like
so:

~~~~ {.java}
public class Genealogy {
    public static List[Person] ancestors(Person myPerson) {
       Collection<Person> result = new LinkedList<Person>();
       Person father = p.getFather();
       while (father != null) {
           result.add(father);
           father = father.getFather(); 
       }
    }
}
~~~~

It looks pretty and neat. Mostly because the example is simple and because my
solution is correct. Let's try a faulty version:

~~~~ {.java}
public class Genealogy {
    public static List[Person] ancestors(Person myPerson) {
       Collection<Person> result = new LinkedList<Person>();
       Person father = myPerson;
       do {
           father = father.getFather(); 
           result.add(father);
       } while (father != null);
    }
}
~~~~

You see what happens in the above version if `myPerson.getFather()` is `null`?
Yes, it crashes and yes, it's a bug. The issue with `null` is that nobody will
warn you in this case. You'll have to wait for the tests (the best case) or the
bug (the worst case) to see it, thanks to a wonderful null pointer exception.

Ok, move on to Haskell:

~~~~ {.haskell}
ancestor (Person _ _ _ Nothing ) = []
ancestor (Person _ _ _ (Just p)) = p:(ancestor p)
~~~~

So, as you might have noticed, I propose a recursive definition. It can be read
as: if no father is defined, returns the empty list. Otherwise return a list
made of the content of the father field and the list of the father's ancestors
(`:` is the prepending operator in Haskell). As we have to unbox the content of
father to get the person's value, we can't do it wrong. Typing saves you from
using a *null* value. Thanks to the "Maybe boxing" we have a type safe version.

So `Maybe` provides you a convenient solution to avoid some bugs, catching
errors at compile time instead of doing it at runtime. Unfortunately, as old
Lulu used to say, "you can't spend good time with me and keep your 200 bucks".
Here type safety has a performance price. we will see later that the `Maybe`
type is very convenient to use in many scenarios, but that's enough for today.

# Haskell extra point

Here, I would like to introduce some precision about the non-`null`policy and
boxing issues. We agree that boxing might be painful when you know that
something will suceed. You often want to let it go. You can but not with `null`.
Instead, we have exceptions. The idea is that if you don't provide the expected
value to a function, the function may throw an exception. Yes, we don't have a
typesafe failure anymore. Hey, that's what you asked for! The main difference
here is that: we won't silently continue with the non-existing value. We just
stop the computation and throw an error.

As an example, you can have a look at the standard `head` function that returns
the first element of a list and how it behaves on a non-empty list:

~~~~
Prelude> head [1,2]
1
~~~~

and on an empty list:

~~~~
Prelude> head []
*** Exception: Prelude.head: empty list
~~~~

[beust]: http://beust.com/weblog/2012/08/19/a-note-on-null-pointers/ 
