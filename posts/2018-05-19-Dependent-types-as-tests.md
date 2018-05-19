---
title: "Dependent types as tests"
lang: en
category: [English, Idris, Functional Programming]
date: 2018-05-19
author: Nicolas Biri
---

Types are not supposed to replace tests… in general.
However, dependent types allow you to express tests as types, allowing you to
run your tests automatically at compile times.
Even better, you can express property tests and prove them rather than test them
on a large subset of values.

To be accurate, we're not talking about tests here, rather of proofs. I'm using
_tests_ just because what we're checking can be compare to what we check with
usual tests.
Please don't mind if I don't use the appropriate term in favor of a more
informal terms, I just want to prove people that we can replace tests with a
better alternative, with the appropriate language.

This article illustrates this possibility through the [tennis kata].
Our objective, provide a way to compute and display a tennis score (of only
one game) correctly, and prove that our code is right.
Let's see how "if it compiles, you can ship it" can be real.

# The tennis kata with dependent types

# Types, types, types

First, let's write some idris to encode the score progression and scores display.

To encode the score, we use the following rules. To win a game, a player must:
- have scored at least four points;
- **and** must be ahead by at least two points.

We start by the obvious, we need two players:

```idris
data Player = Player1 | Player2
```

A classical way to represent score is to use two naturals.
This encoding however, allows a lot of invalid combination.
For example, 5-0 is not a valid score, as the previous score,
4-0, should already have led to the end of the game.

A way to tackle this with dependent type is to embed in a score a proof that
the game is not over.
Let's build a type that can prove that nobody has won yet:

```idris
||| Prove that the game is not over after a point
data NotWinning : (winnerScore: Nat) -> (loserScore: Nat) -> Type where
  ||| Game is not over because the winning player is below 40
  ThresholdNotMet : LTE x 3 -> NotWinning x y
  ||| Game is not over because we only reach deuce or advantage
  OpponentTooClose : LTE x (S y) -> NotWinning x y

```

The first constructor is a proof that nobody has won because the winner of a
point has not reached 4 points yet (they have less or equal to 3 points).
The second constructor is a proof that the winner of a point is not 2 points
ahead yet.

Now that we have a type for our proofs, let's write a decision procedure to
build a proof or to prove the lack of proof that a game is over depending on a
score:

```idris
||| Decision procedure to obtain a NotWinningProof
notWinning : (winnerScore : Nat)
              -> (loserScore : Nat)
              -> Dec (NotWinning winnerScore loserScore)
notWinning winnerScore loserScore =
  case isLTE winnerScore 3 of
       (Yes prfT)    => Yes (ThresholdNotMet prfT)
       (No contraT) => case isLTE winnerScore (S loserScore) of
                            Yes prfO => Yes (OpponentTooClose prfO)
                            No contraO => No (winningProof contraT contraO)
  where
    winningProof : (contraT : Not (LTE winnerScore 3))
                -> (contraO : Not (LTE winnerScore (S loserScore)))
                -> Not (NotWinning winnerScore loserScore)
    winningProof contraT contraO (ThresholdNotMet x) = contraT x
    winningProof contraT contraO (OpponentTooClose x) = contraO x
```

We check each condition that are necessary to build one of our constructors.
If one of the two condition is met, we are able to provide our proof, otherwise
we use the two witnesses that our conditions aren't met to build a proof that we
can not build a witness that the game is not over (we have a winner).

## Score

From there, we are able to define a type for the score of an ongoing game:

```idris
||| Score is built by stacking ball winners, along with the proof that the game is not over
data Score : Nat -> Nat -> Type where
  Start : Score 0 0
  Player1Scores : Score x y -> {auto prf: NotWinning (S x) y} -> Score (S x) y
  Player2Scores : Score x y -> {auto prf: NotWinning (S y) x} -> Score x (S y)
```

Our score type does not exactly handle the score. It's rather a list of points
winners, tied with the proof that the game is not over.

We're almost ready to implement the function that allows a player to score a
point.
Before, we need a helper to compute the next score, given a winner and a player,
we compute the hypothetical next score of this player (if the game is not
over):

```idris
nextScore : (winner : Player) -> (pointsOwner : Player) -> (points : Nat) -> Nat
nextScore Player1 Player1 points = S points
nextScore Player1 Player2 points = points
nextScore Player2 Player1 points = points
nextScore Player2 Player2 points = S points
```

And here we go with the scoring:

```idris
||| Add the result of a point to a score
score : (previousScore : Score p1Points p2Points)
      -> (p : Player)
      -> Maybe (Score (nextScore p Player1 p1Points)
                      (nextScore p Player2 p2Points))
score previousScore p {p1Points} {p2Points} =
  case notWinning (winnerPoints p) (loserPoints p) of
       Yes _      => pure (builder p)
       No  contra => 
  where
    winnerPoints : Player -> Nat
    winnerPoints Player1 = S p1Points
    winnerPoints Player2 = S p2Points
    loserPoints : Player -> Nat
    loserPoints  Player1 = p2Points
    loserPoints  Player2 = p1Points
    builder : (p : Player)
    -> {auto prf : NotWinning (winnerPoints p) (loserPoints p)}
           -> Score (nextScore p Player1 p1Points) (nextScore p Player2 p2Points)
    builder Player1 = Player1Scores previousScore
    builder Player2 = Player2Scores previousScore
```

The `score` function, given the winner of a point, will either returns the next
score or nothing, if the winner of the point won the game. The code almost
entirely relies on the `notWinning` function, which provides either the proof
needed to build the next score, or the proof that we have a winner.

Then, we can replay a whole game quite easily with `foldlM`:

```idris
||| Replay a full list of balls, if the game ends, the remaining balls are discarded
replay : List Player ->  Either Player (x' ** y' ** Score x' y')
replay = foldlM f (_ ** (_ ** Start))
  where
    f : (a ** b ** Score a b) -> Player -> Either Player (x' ** y' ** Score x' y')
    f (_ ** _ ** s) p = maybe (Left p)
                              (pure . (\s' => (_ ** _ ** s')))
                              $ score s p
```

The result of a replay is either a winner, or an existential type (or Σ-type)
with the current score. The existential type is required since we do not know in
advance what the score will be.

## Display

We're almost there, we just need a few facilities to display the score:
We start by a helper to display points in the usual and weird tennis style.
The points are announced only for the 3 first points of a player (before, the
*deuce* and *advantage* part), thus we need a proof that we are less or equal to
3 to display the points:

```idris
displayPoints : LTE x 3 -> String
displayPoints LTEZero = "0"
displayPoints (LTESucc LTEZero) = "15"
displayPoints (LTESucc (LTESucc LTEZero)) = "30"
displayPoints (LTESucc (LTESucc (LTESucc LTEZero))) = "40"
```

And we can know display the whole score, we start with the special cases and
then the general one:

```idris
displayScore : Score x y -> String
displayScore {x = Z} {y = Z} _ = "love"
displayScore {x = S (S (S Z))} {y = S (S (S Z))} _ = "deuce"
displayScore {x} {y} _ = case (isLTE x 3, isLTE y 3) of
  (Yes prfX,   Yes prfY)   => concat [displayPoints prfX
                                     , " - "
                                     , displayPoints prfY
                                     ]
  _                        => case compare x y of
                                   LT => "advantage Player2"
                                   EQ => "deuce"
                                   GT => "advantage Player1"
```

# Tests

Isn't this article supposed to be about testing? Well… here we are.

## Unit testing

Let start with something simple. If we provide to `replay` an empty list of
winner, the score must be unchanged.
A usual way to test it in a language with dependent type would be something
like:

```idris
||| pseudocode, does not compile
replayEmptyListGivesStart : IO Bool
replayEmptyListGivesStart = assertEqual (Right (0 ** 0 ** Start)) (replay [])
```

Well, we can actually do better:

```idris
replayEmptyListGivesStart : Right (0 ** 0 ** Start) = replay []
replayEmptyListGivesStart = Refl
```

This is a compile time test.
We have expressed here that `replayEmptyListGivesStart` is an element of
the structural equality between `Right (0 ** 0 ** Start)` and `replay []`.
We are even able to provide this element which is `Refl`. Behind the hood,
the compiler will rewrite `replay []`, be able to reduce it
`Right (0 ** 0 * Start)`. And as `Refl` is of type `a = a`, it's a correct value
for this type and our program compiles!

We've just embedded in our program a _proof_ that `Right (0 ** 0 ** Start) = replay []`.
This proof will be run at each compilation, we can't ship our code if this is
not true anymore.

What would happen if it is the case?

Let's try and change the type of `replayEmptyListGivesStart` with the following:
`replayEmptyListGivesStart : Left Player1 = replay []`

We obtain this error message:

```
/Users/biri/Perso/katas/Tennis/TennisDT.idr:133:29-32:
    |
133 | replayEmptyListGivesStart = Refl
    |                             ~~~~
When checking right hand side of replayEmptyListGivesStart with expected type
        Left Player1 = replay []

Type mismatch between
        Right (0 ** 0 ** Start) = Left Player1 (Expected type)
and
        x = x (Type of Refl)

Specifically:
        Type mismatch between
                Left Player1Unification failure
        and
                Right (0 ** 0 ** Start)
```

The compiler complains that we provide an element of type `x = x` while it was
able to compute `Left Player1 = Right (0 ** 0 ** Start)`.
The output is quite precise about what went wrong here.

Similarly, we can test some specific results of our display function:

```idris
displayStartIsLove : "love" = displayScore Start
displayStartIsLove = Refl

display1stWinnerGot15 : (s : Score 1 0) -> "15 - 0" = displayScore s
display1stWinnerGot15 _ = Refl

```


## Property testing

Let's now test that our replay function got a love game right.
A love game happens when a player won the 4 first points in a row,
ending the game without leaving any point to their opponent.

Let write a simple function to test this:

```idris
testLoveGame : Left p = (p : Player) -> replay (replicate 4 p)
```

Given a player, whoever it is, we can prove that this player won the game when
he won the 4 first points
A first try to implement this function might be:

```idris
testLoveGame : (p : Player) -> Left p = replay (replicate 4 p)
testLoveGame p = Refl
```

Unfortunately, it doesn't work. And I won't provide you the error here, because
it's quite (very) verbose.
The problem is that without knowing the value of `p` (the wining player), the
compiler isn't able to reduce the left term of the equality. Fortunately,
splitting cases for the different values of `Player` is enough:

```idris
testLoveGame : (p : Player) -> Left p = replay (replicate 4 p)
testLoveGame Player1 = Refl
testLoveGame Player2 = Refl
```

Knowing the value of `p`, the compiler is now able to reduce the left term up to
`Left p`, and thus to typecheck.

With the same strategy, we can check more advance properties, like ensuring that
one player can't win from deuce or that one player win if he scores two times
after a deuce:

```idris
testCantWinFromDeuce : (p : Player) -> True = isRight (replay (take 11 $ cycle [p, opponent p]))
testCantWinFromDeuce Player1 = Refl
testCantWinFromDeuce Player2 = Refl

gainWith2PointsFromDeuce : (p : Player) -> Left p = replay ((take 10 $ cycle [Player1, Player2]) <+> replicate 2 p)
gainWith2PointsFromDeuce Player1 = Refl
gainWith2PointsFromDeuce Player2 = Refl
```


## Property testing with recursions

In the last examples, we used a arbitrary number as a parameter for `take`.
Unfortunately, for these examples, testing the property for every number of
points is  bit complex.
To illustrate how we can prove things on natural, let's move to the `display`
function again.

We will need a few helpers first, that are not in the base library:

```idris
compareToSuccIsLT : (x : Nat) -> LT = compare x (S x)
compareToSuccIsLT Z = Refl
compareToSuccIsLT (S k) = compareToSuccIsLT k

compareSameIsEq : (x : Nat) -> EQ = compare x x
compareSameIsEq Z = Refl
compareSameIsEq (S k) = compareSameIsEq k

compareToPredIsGT : (x : Nat) -> GT = compare (S x) x
compareToPredIsGT Z = Refl
compareToPredIsGT (S k) = compareToPredIsGT k

LTnotEQ : (LT = EQ) -> Void
LTnotEQ Refl impossible

LTnotGT : (Prelude.Interfaces.LT = GT) -> Void
LTnotGT Refl impossible

EQnotGT : (EQ = GT) -> Void
EQnotGT Refl impossible

DecEq Ordering where
  decEq LT LT = Yes Refl
  decEq LT EQ = No LTnotEQ
  decEq LT GT = No LTnotGT
  decEq EQ LT = No (negEqSym LTnotEQ)
  decEq EQ EQ = Yes Refl
  decEq EQ GT = No EQnotGT
  decEq GT LT = No (negEqSym LTnotGT)
  decEq GT EQ = No (negEqSym EQnotGT)
  decEq GT GT = Yes Refl
```

The helpers give us two things:

1. the result of a comparison of a natural with it's predecessor, itself, its
   successor;
2. a structural proof of equality/difference between two `Ordering`

With these helpers, we will prove the display of advantages and deuce when a
player is at most one point ahead of their opponent and each player has at least
scored 3 times.
To do so, we must find an implementation for these 3 functions:

```idris
displayDeuce : (x : Nat) -> (s : Score (3 + x) (3 + x)) -> displayScore s = "deuce"
displayAdvantageP1 : (x : Nat) -> (s : Score (S (3 + x)) (3 + x)) -> displayScore s = "advantage Player1"
displayAdvantageP2 : (x : Nat) -> (s : Score (3 + x) (S (3 + x))) -> displayScore s = "advantage Player2"
```

I'm detailing the first one here, the two others follow a similar pattern and
are left as exercises to the reader.
If you look back at display, you can notice that we don't use the value of the
score, only it's type is used to to compute the display.
As a consequence, we don't need to decompose score.
Let's try with what we've learn so far and decompose the first parameter:

```idris
displayDeuce : (x : Nat) -> (s : Score (3 + x) (3 + x)) -> displayScore s = "deuce"
displayDeuce Z _ = Refl
displayDeuce (S k) _ = Refl
```

Of course, we can't decompose the parameter for all the possible naturals.
So we use the decomposition for each possible constructors.
Unfortunately, it's not that easy. Here is the error we obtain with this try:

```
Type mismatch between
        "deuce" = "deuce" (Type of Refl)
and
        case Prelude.Nat.Nat implementation of Prelude.Interfaces.Ord, method compare n
                                                                                      n of
          LT => "advantage Player2"
          EQ => "deuce"
          GT => "advantage Player1" =
        "deuce" (Expected type)
```

The compiler is not able to reduce `compare`. Here is the tricky part.
The solution is thus to match against the result of compare, with a `with` rule:

```idris
displayDeuce : (x : Nat) -> (s : Score (3 + x) (3 + x)) -> displayScore s = "deuce"
displayDeuce Z _ = Refl
displayDeuce (S k) s with (compare k k) proof p
  displayDeuce (S k) s | LT = ?lt_case
  displayDeuce (S k) s | EQ = Refl
  displayDeuce (S k) s | GT = ?gt_case
```

While we know that `compare k k` will always return `EQ`, the compiler has no
way to figure it out.
For the `EQ` case, knowing the result allows the compiler to achieve the
reduction and we can conclude with `Refl`.
The `LT` case and the `GT` case are symmetric.
In both case, we have to prove that the case can't happen.

Here is the information we have for `?lt_case`:

```idris
  k : Nat
  s : Score (S (S (S (S k)))) (S (S (S (S k))))
  p : LT =
      Prelude.Nat.Nat implementation of Prelude.Interfaces.Ord, method compare k
                                                                               k
--------------------------------------
lt_Case : "advantage Player2" = "deuce"
```

So we need to find an element of type `"advantage Player2" = "deuce"` with a
context that contains one very interesting element: `p` that is a "proof" that
`LT = compare k k`

When you want to prove that something is not possible, you basically want to
obtain a value of type `Void` and then conclude with the `void` function,
that can build any type out of `Void`.
`Void` is the bottom type, the one that is uninhabited. We have already some
element to build such type.
`LTisNotEQ` that we have introduced earlier, is of type `Not (LT = EQ)`.
This type is actually an alias for `LT = EQ -> Void`.
And with `p`, we're almost there, we just need to replace `compare k k` with
it's true value, `EQ`, we're just one rewriting rule away.
With the same reasoning for the GT case we obtain the following code for
`displayDeuce`:

```idris
displayDeuce : (x : Nat) -> (s : Score (3 + x) (3 + x)) -> displayScore s = "deuce"
displayDeuce Z _ = Refl
displayDeuce (S k) s with (compare k k) proof p
  displayDeuce (S k) s | LT = void (LTnotEQ (rewrite compareSameIsEq k in p))
  displayDeuce (S k) s | EQ = Refl
  displayDeuce (S k) s | GT = void (negEqSym EQnotGT (rewrite compareSameIsEq k in p))
```

And here we go, it compiles and we have proven that if each players has scored
at least 3 points and have the same number of points, their score will be
"deuce".

This short example also shows the limit of using types as tests. You're doing
more than testing, you are proving facts, which means that you must assist the
compiler to find its way through the code to build the proof.

# A few hints

Let's recap some of useful hint if you want to use types as tests:

1. Unit tests are the easiest, as most of the time, the compiler will be able
   to reduce the full expression if you provide all the required values.
2. When you do property testing (proving actually), you almost always have to
split cases for the variables, to ease the progression of the computer.
3. When the compiler cannot reduce a value, help him with rewrite rules.
   Sometimes, it may be useful to tweak your test a bit to ease the reduction of
   a formula.
   For example, `(x : Nat) -> (s : Score (3 + x) (3 + x)) -> "deuce" = displayScore s`
   is easier to prove than `(x : Nat) -> (s : Score (x + 3) (x + 3)) -> "deuce" = displayScore s`
   because of the implementation of `(+)` for Natural.
4. The error messages will help you to figure out where the compiler is stuck and
   what expression should be rewritten.
5. If the compiler explores cases that are not possible help it to figure it out
by producing a `Void` value.

Hope it helps. Unit testing is usually as easy with types than with traditional
test functions, property testing is usually harder, as it requires you to build
a proof rather than just running the test on a limited set of values.

Have fun coding.

# Acknowledgments

I would like to thanks the Lambdada community, who helped me to figure out how
great the tests as type feature is, have heard the first version of my
explanations and made some remarks to improve the code.

This blog has no comments, for any question or remark, feel free to contact me
on twitter ([@BeRewt])

[tennis kata]: http://codingdojo.org/kata/Tennis/
[@BeRewt]: https://twitter.com/BeRewt
