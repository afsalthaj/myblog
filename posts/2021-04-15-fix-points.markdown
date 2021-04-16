---
title : Fix point types in Scala. A quick brain teaser
---

Let's do some math.

If we continuously do `cosine` of 0 and keep on applying `cosine` over the result, we end up in a number finally
that gets `fixed` forever. The number is 0.73908513321516067.

```scala

  cos(cos(cos(cos(x))))


```

Ofcourse this example is a blatant copy from one of the
most wonderful blogs ever written about fix-points in Scheme language - https://mvanier.livejournal.com/2897.html.

You can choose to read it. But this blog slightly takes a different turn for explaining things to reach the same goal.


Hence the fix-point of `cos` is `x` if `cos(x) = x`. But a fix-point may not be always a real number. It could be even a `type`. 
A `function` is a `type` indeed.

Sure what does that mean? Well if `fix-point` of a function like `cos` is a `real-number`, and if you say that a `fix-point` can be a `function` itself,
that means the following:

```

There exists a `function`, whose fix-point can be a `funtion` itself. Hmmmmm. 

```

## Stop explaining. Show an example.

A factorial of a number 3 is 3 * 2 * 1. It's implementation is as follows:

```scala

  val factorial: Int => Int = 
    n => if (n == 0) 1 else n * factorial(n - 1)

```

Sure, that's easy. But let's say someone asked you to implement the same `factorial` such that the name of the function
shouldn't come in the body. Before you ask `why`, here is a question for you. Why are we implementing `sort` in an interview
while most of us (that I have met) never had to implement a sort ever in our professional career? 

Consider it as a challenge and move on with the rest of the blog.

Answering the question, well may be I can do

```scala

  def factorial(f: Int => Int): Int => Int = 
    n => if (n == 0) 1 else n * f(n - 1)

```

Ah, sure, but that's not a factorial.

Ofcourse, that's not a factorial, so let's call it as `almostFactorial`.

```scala

   def almostFactorial(f: Int => Int): Int => Int = 
     n => if (n == 0) 1 else n * f(n - 1)

```

Now we need to implement a sensible `factorial` in terms of `almostFactorial`. 
All of us know, the real factorial exists when the `f`( now staying as an argument in `almostFactorial`) is `almostFactorial` itself.

i.e, 

```scala

  val factorial: Int => Int = 
    almostFactorial(almostFactorial(almostFactorial(.....)))

```

Well if you try to implement it, you will be writing it forever. Anyone who is not convinced yet, please try it out.

Fine, let's take a different approach then.

```scala

 val factorial0: Int => Int = 
   almostFactorial(identity)

```

```scala

  factorial0(0) // returns 1 ==> works
  factorial0(1) // doesn't work, coz identity(n - 1) == identity(1 - 1) == identity(0) == 0
```

So `factorial0` works only for zero. That's fine for now.

Let's implement for factorial1.


```scala

 val factorial1 = 
   almostFactorial(factorial0) // that works

 val factorial2 = 
   almostFactorial(factorial1) // that works too.

 val factorial2_ = 
   almostFactorial(almostFactorial(almostFactorial(identity))) // that works too.

 val factorialInfinity: Int => Int = 
      almostFactorial(almostFactorial(almostFactorial.............almostFactorial(.........))) // forever

```

Ah, kind of making sense now. `factorialInfinity` is a function, which is a fixpoint of `almostFactorial`.
Infact `factorialInfinitiy` is our real `factorial` implementation.

```scala

 factorial = fix-point-of almostFactorial

```

Hmmm. I think its time to figure out what is this `fix-point-of`. Is there a function called `fix-point-of` that allows us to pass
`almostFactorial` and return a real `factorial`. In other words, is there a function called `fix-point-of` that takes a function
and returns the fix-point of that function.


```

def fixPointOf(f: <someFn>): fixPointOf_SomeFn

```

This `fixPointOf` is our famous y combinator. The `Y` combinator is the one that converts a function to its fix-point.


```scala

 val factorial = almostFactorial(factorial)

 implies,
 
// fix-point-fn = almostFactorial fix-point-fn


```

This implies

```scala

 // fix-point-fn = almostFactorial fix-point-fn
 // Y takes a function and converts to its fixpoint
 // Y (f) = ??? // returns fix-point
 // But, fix-point-fn = almostFactorial fix-point-fn
 // Y (f) = f( Y (f))

  def Y[A](f: A => A) = f(Y(f))
  val factorial = Y(almostFactorial) // Stack overflow haha

```

Stack overflow is, when you call Y the following thing happens:


```scala
 
 f(Y(f)) == 
   f(f(f(f(f.......))))
  

```
In other words, it tries to compute the fix-point function for you and it never ends. 

Lambda saves us from this stack overflow - in a surprising way.

See the below example of a lambda

```
def f(a: Int): Int = a

def g = Int => f(a)

// Here f is actually equal to g, except that `g` is another function that wraps `f`.
g(1) == f(1) == 1

```

Let's do that in our Y combinator. Oh well, why am I calling it as a `combinator` now. 
A combinator by definition should be a lambda that takes 1 argument and its body should never have any free variables,
but only the one defined in the lambda. `lambda x : x + 1` is a combinator, and `lambda x: x + y` is not. `lambda x : ...` is a python syntax
but you know the idea. Forget about this for the time being.


```scala
  def Y[A](f: A => A) = f(a => Y(f)(a))
  // same as def Y[A](f: A => A) = f(Y(f)) except that the latter produces stack overflow. See for yourself why.

```

Now my factorial implementation is

```scala

 val factorial = Y(almostFactorial)

 // factorial(3) is 6
```

Hurray, this blog is incomplete, and yet to be cleaned up. Thanks for your patience. Catch you soon!