---
title : Squaring is monad join
---

How do you implement square (or double) using Monads?

Quite simple: Use the monad instance of a function and then `join` on a function that returns a function.

How on earth does that work?

You know the signature of Monad join.

``` haskell

  join :: Monad f => f (f a) -> f a
  join ffa = ffa >>= id


```

Consider `data Optional a = Full a | Empty` has a monad instance. This implies

``` haskell

  >> join (Full (Full 1))
  Full 1


```

In this case is `f` is `Optional`, and `ffa` is `Full (Full 1)`.
When you replace `f` to be a function `f :: t -> a`, then `ffa` is `f :: t -> (t -> a)`.

An example is :

``` haskell

  >> ffa = \x -> (\y -> x * y)
  >> ffa 2 3
  6
  >>


```

`ffa = \x -> (\y -> x * y)` is equivalent to just `ffa = (*)`.


Thinking along the same lines, calling `join` on `ffa` returns `fa` means converting `t -> (t -> a)` is `t -> a`.

Let's do that

``` haskell

  >> :type (join *)
  (join ffa) :: Num a => a -> a


```

And,

``` haskell

 >> join * 10
 100

```


This implies

``` haskell
  >> square = join (*)
  >> double = join (+)


```
Yea, that was more of a fun. But you can sort of guess that monad instance of a function `(->) t` involves passing the same argument twice to a binary function.
