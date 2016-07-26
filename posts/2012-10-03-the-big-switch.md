---
title: The big switch
date: 2012-10-02 17:00
comments: true
categories: [English, Make it functional]
tags: [Haskell, Java, pattern matching, switch]
---

Pattern matching is often put forward as a first feature example when we
introduce functional programming. And there is maybe a first misunderstanding
there. Actually, when we talked about pattern matching, we *don't* talk about
regular expressions. No, pattern matching in functional programming is
closer to a `switch case`… on steroid. The kind of steroids that
gives to Suzanne Boyle the opportunity to be faster than Usain Bolt.

Java programmers have started to love it with primitive types, then with
[enumeration][switchEnum] and more recently with [`String`][switchString].
In many situations, it allowed them to refactor their code in a more
comprehensive way. Do you think we can go further? The short answer is: Yes.

## Say "Hi!"

It's our first example. In this blog, I will use most of the time toy examples.
The goal is to add as few noise as possible around the concept I will try to
introduce.

Here is the scenario. We deal with people. A person has a first name, a last
name and an age. Yes, welcome back to CS 101. Yes, it is stupid to store the age
as an integer and not as a date, I never said my examples would be clever.
At this stage neither the Haskell nor the Java version are complex. Here
is the Java version:

~~~~ {.java}
public class Person {

    private final String firstname;
    private final String lastname;
    private final int    age;

    public Person(String myFirstname, String myLastname, int myAge) {
        this.firstname = myFirstname;
        this.lastname  = myLastname;
        this.age       = myAge;
    }

    /* Forgive me, I won't write getters/setters.
     * I'm a poor lazy functional programmer.
     */
}
~~~~

And there is the Haskell one:

~~~~ {.haskell}
data Person = Person { firstname :: String
                     , lastname  :: String
                     , age       :: Int
                     }
~~~~

May be I should explain a few things here. Haskell is definitely not object
oriented. As a consequence, the field names given in the `Person` definition
are actually function names to access these fields. For example, if you have a
value `tom` that contains a `Person`, you can write `age tom` to access `tom`'s
age. I know,

Ok, so far, so good. Haskell is more concise, but who cares? Everybody loves
typing, right? And if you don't, you have these lovely code generation features.
Actually, I don't like them but it's a matter of taste.

Suppose now that we want to greet a person. The function `greet` should return
"Hi, " followed by the person full name. Unfortunately we work in a naughty
environment and we want to protect young people (below 18). To them we'll
answer "Thou shall not pass". Except for this little guy, [Isaac McMillen][boi],
because we don't really care if he gets hurt. For him we have a special message:
"Welcome home, Isaac".

## Check fields, write too many things

Let's build our Java greeting class (we can as well add a `greet` function to
the `Person` class but it doesn't really match the single responsibility
principle)! To do so I assume that you've added a `getFullname` function to the
person class. I can leave it to you as an exercise, you're allowed to find it
insulting.

~~~~ {.java}
public class Greetings {
    public static String greets(Person myPerson) {
        if (isIsaac(myPerson) 
             return "welcome home, Isaac";
        else if (isOverEighteen(myPerson))
             return "Hi, " + myPerson.getFullname();
        else
             return "Thou shall not pass";
    }

    /* Helper functions */

    private static boolean isIsaac(Person myPerson) {
        return myPerson.getFirstname().equals("Isaac")
            && myPerson.getLastname().equals("McMillen");
    }

    private static boolean isOverEighteen(Person myPerson) {
        return myPerson.getAge >= 18;
    }
}
~~~~

I know you, sweet little troll! You wanna talk about multiple returns, string
concatenation, parenthesis and code convention. It's not the point here! We
don't want to talk about it at the moment. So relax and stay focus on the
algorithmic part.  We have an `if … else if` construct, and we have a lot of
access to the fields.  Look especially at the `isIsaac` function. It's so
cumbersome! And we just want to test two fields! Let's compare it to the Haskell
version:

~~~~ {.haskell}
greet :: Person -> String
greet p = 
  case p of 
    (Person "Isaac" "McMillen" _) -> "Welcome home, Isaac"
    (Person _ _ age)
      | age < 18                  -> "Thou shall not pass"
      | otherwise                 -> "Hello " ++ (fullname p)
~~~~

It's denser, yes. The first line give us the type of the function (it
takes a person, it returns a string). We can as well omit it, since Haskell is
able to work out the type on its own. But look how we have separated the three
cases. This is pattern matching in action. Instead of the painful `isIsaac`
function, we have a clean pattern matching, that inspect the fields of our
person. Similarly we use the same trick to extract the age field. It's a minor
improvement, but most of the functional programmers find it quite interesting.

How many time did you extract some fields of a variable to inspect them?
Pattern matching let you to do it in an efficient way, without having to
struggle with accessors.

I know it might seem a minor improvement. You can even think it's not an
improvement at all. As I claimed in my previous post, you don't have to be
convince that functional programming is better. But, for us, it's full of those
kind of minor change that simplify our work as developers, and I hope it helps
you get into it.

## Haskell bonus point

Haskell folks love pattern matching so much that they allow you to use it
directly in the function definition. Like so: 

~~~~ {.haskell}
greet :: Person -> String
greet (Person "Isaac" "McMillen" _) = "Welcome home, Isaac"
greet (Person f l age) | age < 18   = "Thou shall not pass"
                       | otherwise  = "Hello " ++ (fullname (Person f l age))
~~~~

If you compare it to the first version, you can notice that the last case is
more verbose. With this version, we had to rebuild a Person to obtain its
fullname. How stupid is this?! Well Actually, the stupid one isn't Haskell in
this case, it's me, since pattern matching can do better than the previous
version:

~~~~ {.haskell}
greet :: Person -> String
greet (Person "Isaac" "McMillen" _)   = "Welcome home, Isaac"
greet p@(Person _ _ age) | age < 18   = "Thou shall not pass"
                         | otherwise  = "Hello " ++ (fullname p)
~~~~

The part `p@(Person _ _ age)` means that we're looking for a person, we assign
this person to the `p` value and we assign the person age to the `age` value.
Concise, efficient, functional.

Pattern matching is so useful in functional programming that we often don't use
accessors present in our datatype. A straightforward consequence is that we
often omit the accessors wen we define types. For example, another possible
definition of `Person` might be: `data Person = Person String String Int`. Short
enough?

[switchEnum]: http://docs.oracle.com/javase/1.5.0/docs/guide/language/enums.html
  (Java includes switch for enum in java1.5)
[switchString]: http://docs.oracle.com/javase/7/docs/technotes/guides/language/strings-switch.html
  (Java includes switch for string in Java 7)
[boi]: http://en.wikipedia.org/wiki/The_Binding_of_Isaac_%28video_game%29 
  (The main character of the wonderful game "Binding of Isaac ")
