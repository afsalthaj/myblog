---
title : Monad Transformers
---

## Monad Transformers in Scala

This is a simplified explanation of Monad transformers in Scala. I highly recommend you reading my blog
on "Birth of Monad Transformers through Compose" in Haskell once you get a gist of what monad transformer is from this blog.

Let’s start with a question.

Do you think the type List is a Monad? Yes, it has got a bind (flatMap) and unit (apply) method that
satisfy the requirements to be monad + it follows monad laws. The monad list can also be termed as an effect
because “given a type A, lifting it (somehow) as List adds new capabilities to it, such that, we can now aggregate over A,
find the sum of A and so on and so forth. The type A never had these capabilities when it was existing by itself, until
we lifted it to become a List. That was a deep dive straight away. Let’s take a breath now and say — Well, if you lift your type to a Monadic Effect,
it now possess more capability. Get into a spaceship and say now YOU are capable of moving to space — Spaceship Monad!

Similarly, when we lift A to the effectOption, we say now A has another capability —
A can have a defined absence — None. Now, we never want to throw new Exception to mark the absence of A.

Let’s lift the type A to another effect, scala.util.Try. Now A can be a Failure or a Success and not just A.
The type A inherently has the capability to handle failure. I hope, you almost got the point now.
This is more or less a conceptual understanding of what effects are, and we are naming it to be a “Monad”. Pause here and play with Monads.


### Effect from a different angle

From another angle, we say any computation that should potentially yield A would become more intuitive and useful if it is yielding `A` under an effect, instead of a raw `A` .
A typical example is trying to get an account from an account repository by passing an Id. If the computation has a return type of Account, it means, it should always exist.
We want the computation’s return type to expose the fact that, account may or may not exist.
In other words, it should return an Account under an effect — Option. Hence you may find the second get function in the below example to be more sensible.


``` scala

  def get[Account, Id](id : Id): Repository => Account = ???
  def get[Account, Id](id: Id): Respository => Option[Account] = ???


```

Was that effect enough?

Well, we said the repository may or may not provide an account, and hence we returned an Option of account.
We know the computation is under the effect Option. But is that enough to depict the behaviour of a database operation?

What if the operation resulted in a database failure and you want to tell that to the user of the function `get`?.
This simply means the result should be "either" a `database exception` "or" `Option[A]`.

In other words, we need to lift Option[A] to another effect that can handle a database exception.
We can use scalaz’s either \/ (if using cats, you can rely on scala's `Either` for the rest of the blog) to depict this behaviour.
The get function hence returns a DatabaseException \/ Option[A].

At this point you have fairly a complex return type that is basically a stack of two effects — Option and on top of it, \/[DatabaseException, ?].

### Multiple Effects and Monad Transformers.

You are almost convinced that any computation can yield effects that are stacked upon each other similar to `DatabaseException \/ Option[Account]`.

However, stacking the effects leads to difficulty in extracting the actual value that we need to further execute the rest of the computation with Account.
We wish the stacked effect was just acting as one single effect — one single Monad, so that we can flatMap over it or map over it and continue the operation; something like this:

``` scala

  trait Repository
  trait DatabaseException

  def get[Account, Id](id: Id): Repository => DatabaseException \/ Option[Account] = ???


```

The only way to make that happen is Monad Transformers . We are saying we need a single effect that depicts the multiple layered effect.
We use scalaz’s `OptionT` monad transformer to depict the effect of returning a value under the effectOption which is in turn, under the effect `\/`


``` scala

  sealed trait Account
  case object SavingsAccount extends Account
  case object InvestmentAccount extends Account


  val optionalAccount: Option[Account] = Some(savingsAccount)

  // Practically this is not how you would end up having a throwable on left. This is just to allign the types.
  val optionalAccountWithEitherEffect: Throwable \/ Option[A] =
    optionalAccount.right


```

``` scala

  type EitherEffect[A] = Throwable \/ A

  val singleLayeredEffect =
    OptionT[EitherEffect, Account] {  optionalAccountWithEitherEffect  }

  singleLayeredEffect.map { account => account.debit } // do some computation with account straight away
  // OptionT[EitherEffect, Account]


```

You might need to stare at it for a while. When you had another effect from thin air wrapping your actual effect,
you somehow want to make sure that you still have the capability to peal off all your effects with a single operation (in this case, map)
and get the value without any sort of nested for comprehensions, and that’s all MonadTransformer does! In the above example, the monad transformer OptionT converts an effect of
`Option` layered with `\/[Throwable, ?]` to one single effect OptionT that makes you feel like it is still an option and you are flatMapping over it.


### Mechanism behind Monad Transformers (Optional Read)

For those who want to know the mechanism behind monad transformers, you can consider it as composing monads, which is in fact is a difficult thing to achieve if we have.

We have a type called Compose that can be constructed using two type constructors f and g. This means kind of compose is `(* -> *) -> (* -> *) -> (*) -> *`

Let's spit out a few Haskell here because it's always good to get
around understanding these concepts in Haskell.

Let's define `Compose` and try to create instances of `Applicative`, `Functor` and `Monad` for `Compose`

If you are not familiar with the syntax then all you need to understand here is `Compose[F[_], G[_], A]` cannot have an instance of
`Monad` unless we know more about `G[_]`.

``` haskell

  // case class Compose[F[_], G[_], A](value: F[G[A]])
  newtype Compose f g a =
    Compose (f (g a)) deriving (Show, Eq)

  instance (Functor f, Functor g) =>
    Functor (Compose f g) where
      (<$>) f (Compose fga)  = Compose ((\ga -> f <$> ga)  <$> fga)

  instance (Applicative f, Applicative g) =>
    Applicative (Compose f g) where

    pure a = Compose (pure (pure a))
    (<*>) (Compose fgab) (Compose fga) = Compose (lift2 (<*>) fgab  fga)

  -- Its impossible to implement bind, but if we know g is "Maybe" as an example, then its possible.
  -- However when specialising G, it becomes Monad Transformers and not actually Compose.
  -- Compose[F[_], Option, A] <=> OptionT[F[_], A]

  instance (Monad f, Monad g) =>
    Monad (Compose f g) where
    (>>=) = error "Not possible"

```

In our example, which is  `Either[Throwable, Option[A]` (or `Throwable \/ Option[A]`) can be easily lifted to Compose
and do functor (map) operations, and applicative operations, but not monadic operations.

To detail it further, a monad instance of Compose requires defining a bind as given below:

``` haskell

  (>>=) :: (a -> Compose f g b) -> Compose f g a -> Compose f g b


```

or in scala,

``` scala

  def flatMap (f: Compose[F[_], G[_], A], g: A => Compose[F[_], G[_], B]): Compose[F[_], G[_], B] = ???


```

This means you cannot do

``` scala

  val layered1: Either[Throwable, Option[Int]]

  val layered2: Either[Throwable, Option[Long]]

  Compose(layered1).flatMap(_ => layered2)


```

That said, let's don't assume Compose useless for this reason. For instance, it has got a Traversable.

``` scala

  val layered: Either[Throwable, Option[Int]]

  def getUserFromDb(id: Int): IO[Either[Throwable, Option[User]]

  val whatWeWant: IO[Compose[Throwable, Option[User]] =
    Compose(layered).traverse(getUserFromDb.map(Compose.apply))


```

This also implies, there isn't always a need of monad transformers to deal with composed effects.

#### Summary

The issue of composing monads is often addressed with a custom-written version of each monad that’s specifically constructed for composition.
This thing is called a monad transformer.

There is something more to it before we conclude.

If you ever tried before to implement general monad composition, then you would have found that in order to implement join for nested monads F and G, you’d have to write a type such as F[G[F[G[A]]]=> F[G[A]] and that can’t be written generally. However, if G happens to have a traverse instance, we can sequence to turn G[F[_]] into F[G[_]], leading to F[F[G[G[A]]]], which in turn allow us to join the adjacent F layers as well as the adjacent G layers using their respective Monad instances. I am not going to explain how, but give it a try.


### Where to go from here ?

It's good to stop here, because you already got the gist of what Monad Transfomres are, and how to use them. Refer to typelevel cats and scalaz documentations
in Google if you are in Scala world. Also, take a look at ZIO and it's libraries that took a different approach towards Monad Transformers in Scala handling performance issues.


#### Hope you find the blog useful in solving your problems.
