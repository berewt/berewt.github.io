---
layout: post
title: Some languages are more equals than others
categories: English, Make it functional
tags: Haskell, Java, equals, typeclass
---

In my last three posts, I've ended my Java example with a comment to indicate
that I won't write extra methods. `equals` is one of them. Today, we're going to
write what we need to test equality of two people.

## Fill in the blank

Today's scenario is really basic, so we can dive directly into the code. We
start with the Person class in Java from the previous post:

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
And the Haskel version:

~~~~ {.haskell}
data Person = Person { firstname :: String
                     , lastname  :: String
                     , age       :: Maybe Int
                     , father    :: Maybe Person
                     }
~~~~

## Painfull equality and useful shortcuts

It is so boring to build equality that most Java IDE have a shortcut for it.
Fortunately we have only one `equals` method to write and the class is quite
short. Of course, we also have to provide an appropriate `hashCode` method, but
as it doesn't change anything to this example, its code is left to the reader.
In general, there is no need to bind the implementation of `equals` and
`hashCode`. It is often seen as mandatory for Java developer because both
function are implemented in `Object`. But you may need to test equality without
providing an hashing function, or the converse. So let's continue with Java
`equals` function:

~~~~ {.java}
 public boolean equals(Object o) {
     if (this == o) return true;
     if (! (o instanceOf Person)) return false;
     Person other = (Person) other;
     return Objects.equal(firstname, other.firstname) &&
            Objects.equal(lastname, other.lastname) && 
            Objects.equal(age, other.age) && 
            Objects.equal(father, other.father); 
}
~~~~ 

`Objects` class was introduced in Java 7 and is very very incomplete at the
moment (in my opinion) but that's out of scope. At least, its introduction allow
us to avoid the use of ternary operators to check nullity. If you use an older
version of Java, you should use stuff like `father == null ? other.father ==
null : other.father.equals(father)` in your code but verbose code is easier to
maintain, right?

Lets do it in Haskell. It's also painful. Aside the type declaration, you have
to claim that you have an _instance_ of equal for your Person type:

~~~~ {.haskell}
instance Eq Person where
  (Person fn1 ln2 age1 father1) == (Person fn2 ln2 age2 father2) =
    fn1     == fn2    &&
    ln1     == ln2    &&
    age1    == age2   &&
    father1 == father2
  _ == _ = False
~~~~ 

Yes, its only a bit shorter. The test of different "fields" is a bit easier.
The main reason is that we don't have the differentiation of pointer equality and
object equality. The only kind of equality that can be defined in Haskell is
object equality. Another reason is that we don't have to bother with `null`
but we have already talked about it. There is a third reason. Try to guess what
it could be, the answer is in the next part of this post.

Haskell could have stick to this solution and wait for solutions provided by IDE
to simplify the life of the developers. Instead, they propose a keyword,
`deriving`. With this keyword, we can add equality (and other typeclasses) to a
type quite easily. In our specific case, we have to add the following line right
below the definition of the Person type: `deriving (Eq)`. This time, it's
shorter:

~~~~ {.haskell}
data Person = Person { firstname :: String
                     , lastname  :: String
                     , age       :: Maybe Int
                     , father    :: Maybe Person
                     }
              deriving (Eq)
~~~~ 

## Type safety everywhere

So, what is the third difference between the "long implementation of equality in
Haskell and its Java counterpart? The short answer is there is no explicit type
check in the Haskell version. The complete answer is there is no type check **at
runtime** in the Haskell version. In Haskell, when you use equal, the compiler
requires that both arguments have the same type. It's not better or worse, it's
different. The motivation for this choice is as follow: Haskell ask you to
know which are the exact types of the values you manipulate. So, if you come
with two values of different types and compare them, it considers it as an
error. Most of the time, it works similarly in Java. Most of the time... We will
see what happens in the other case in a next post.

## Haskell Bonus Point

In the Haskell snippets, where does this `Eq` comes from and what means this
`instance` keyword? `Eq` is what we called a [typeclass]. For any type, you can
provide an instance of a typeclass, providing the function required by it. In
the `Eq` cases two function are defined: `(==)` and `(/=)`. The complete
definition is the following:

~~~~ {.haskell}
class  Eq a  where
    (/=)             :: a -> a -> Bool
    x /= y           =  not (x == y)
    (==)             :: a -> a -> Bool
    x == y           =  not (x /= y)
~~~~ 

As you can see, there is already a minimal definition of `(==)` and `(/=)`,
which allows you to define only one of the two functions when you create an
instance. A first remark about type signature (`(/=) :: a -> a -> Bool` and
`(==) :: a -> a -> Bool`). As often in Haskell, the signature is here for
documentation purpose. It can be read as: function `(==)` (and respectively
`(/=)`) takes an `a` to produce a function that takes an `a` to produce a
boolean. Its a bit different than a function that takes two `a` to produce a
boolean, but we will focus on it another day. Actually, it can be used as if it
was a two `a` argument function. `a` is a type variable, which must be replace
by the correct type when you declare your instance. In our example, it was
replaced by `Person` in our declaration. Some other little Haskell things. The
use of parenthesis around a function with two parameters indicates that you can
use the function without the parenthesis in an infix manner. To test equality of
two values `a` and `b`, you can either write `a == b` or `(==) a b`.

Typclasses can be seen from a Java developer point of view as interfaces. It's
not totally accurate since you can't build a one-to-one mapping between the two
approach. There are some analogies between the imperative and the functional
paradigm, but they fail at some point. I'm giving you two major differences
(there are certainly more of them):

- You can define a default implementation for some method in a typeclass, not in
  an interface.
- You can add instance to a type whenever you want. Not only when you introduce
  it in your code. If my type doesn't have an instance of `Eq`, I can add it
when I need to.

[typeclass]: <http://en.wikipedia.org/wiki/Type_class>
  (Yes, a wikipedia link, the article is quite complete and comprehensive)
