---
title: "Maybe: You want something more?"
comments: true
categories: [English, Make it functional]
tags: [Haskell, Java, Maybe, monad, null]
---

So, you're now convinced that type safety is far better from these `null` things
to avoid insidious bugs. If you're not, read the [previous article][maybe1]
again. So, we have a nice type to avoid nullity, but how do we use it? When we
want to access them, update them, compose them, is it more or less convenient
than using `null`?I could say that [category theory][category] answers this
question. It does, but I guess you came here because you wanted proof.

## Make everything optional

Back to our person data again. Suppose that finally the age of a person is now
optional. But unfortunately we are still going to deal with it when it's
available. Yes, as usual dealing with non-mandatory elements sucks. So here are
our starting point in Java:

~~~~ {.java}
public class Person {

    private final String  firstname;
    private final String  lastname;
    private final Integer age;
    private final Person  father;

    public Person(String  myFirstname,
                  String  myLastname, 
                  Integer myAge,
                  Person  myFather) {
        this.firstname = myFirstname;
        this.lastname  = myLastname;
        this.age       = myAge;
        this.father    = myFather;
    }

    /* Feed it with any expected extra-methods */
}
~~~~

and in Haskell:

~~~~ {.haskell}
data Person = Person { firstname :: String
                     , lastname  :: String
                     , age       :: Maybe Int
                     , father    :: Maybe Person
                     }
~~~~

And you know these data mining guys with their odd queries? Here they are! You
were asked to dig for a person's father's age. Of course, you can only provide
this information if the person's age, father and the father's age are available.
In any other case, you won't be able to provide a better result than `null` or
`Nothing` depending whether you use a poorly designed language or a cool one.

## Running out of the null hell

There are so many optional values that we have a hard time with them, for
example, here is the Java version:

~~~~ {.java}
public class FamilyQuery {
    public static Integer fathersAgeAtBirth(Person p) {
        Integer currentAge = p.getAge();
        if (currentAge == null)
            return null;
        Person father = p.getFather();
        if (father == null)
            return null;
        Integer fathersAge = father.getAge();
        if (fathersAge == null)
            return null;
        else
            return fathersAge - currentAge;
    }
}
~~~~

Hopefully, multiple returns make things a bit easier. Without them, you would
have to used an ugly nested-if structure. The main issue with this
piece of code is that you have to interrupt your navigation to check for
nullity, and then continue with it. Well you know, criticism is easy, can we do
better? You can judge it by yourself, here is the Haskell version:

~~~~ {.haskell}
fathersAgeAtBirth :: Person -> Maybe Int
fathersAgeAtBirth p =
  do
    currentAge <- age p
    pFather <- father p
    fathersAge <- age pFather
    return $ subtract fathersAge currentAge
~~~~

The `do` notation is a way to dive into the content of `Maybe`s. Each time you
use the left arrow, you try to extract the content from right side into the left
side variable. If it isn't possible you stop and return `Nothing`. You don't have
to worry anymore whether a field is defined or not, `Maybe` has the power to
deal with it.

## Haskell bonus point

### For operator lovers (idiomatic Haskell)

I must confess. The above version is a light, sugar free Haskell solution.
Actually, dealing with value enclosed in a structure (here, the `Maybe`
structure) is done many time in a program. Thus, we have some special functions
to do it more efficiently. It can lead to the following code.

~~~~ {.haskell}
fathersAgeAtBirth :: Person -> Maybe Int
fathersAgeAtBirth p =
  subtract <$> fathersAge <*> currentAge
  where
    fathersAge = father p >>= age
    currentAge = age p
~~~~

Yes, most of the time, it is where we lost most of the imperative programmers,
especially those who prefer verbose code with clear, literal function names.
Indeed, perl lovers shouldn't be concerned. I don't have the willpower to explain
it for now, because I would need to introduce many new functions and concepts to do
it right. Just look how the first line of the function `subtract <$> fathersAge
<*> currentAge` looks like what we expect to code if we just have to manipulate
integers. It's a central idea in functional programming: add a concept thanks to
an adequate data structure and provide function to make the introduction of this
concept as little intrusive as possible.

### Back to pattern matching

Remember the first post on pattern matching? We can inpsect a complex data
structure with it. Here, we need to check that:

- the age of the person is defined;
- the father is defined;
- the father's age is defined.

In any other case, we won't be able to produce another result than `Nothing`.
Here is the result:

~~~~ {.haskell}
fathersAgeAtBirth :: Person -> Maybe Int
fathersAgeAtBirth (Person _ 
                          _ 
                          Just(currentAge) 
                          (Person _ _ Just(fathersAge) _) =
  Just (fathersAge - currentAge)
fathersAgeAtBirth _ = Nothing
~~~~

This kind of pattern matching is a bit too complex to be considered as
idiomatic, but it's another illustration of how powerful it can be.

[maybe1]: </English/Make%20it%20functional/2012/10/10/nothing-just-call-me-maybe/>
  "Nothing, Just, call me Maybe (on this blog)"

[category]: <http://en.wikipedia.org/wiki/Category_theory>
  "Please, if you don\'t want to bother with theory, don't click"
