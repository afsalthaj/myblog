---
title : Fix point types and Y combinator - in Scala.
---

Let's do some math.

If we continuously do `cosine` of 0 and keep on applying `cosine` over the result, we end up in a number finally
that gets `fixed` forever. The number is 0.73908513321516067.

```scala

  cos(cos(cos(cos(x))))


```

Ofcourse this example is a blatant copy from one of the
most wonderful blogs ever written about fix-points in Scheme language - https://mvanier.livejournal.com/2897.html.

You can choose to read it. This blog is a mere adoption of the ideas presented in the blog, but in Scala language.

## Fix point being a function

So, yes, the fix-point of `cos` is `x` if `cos(x) = x`. But a fix-point may not be always a real number. It could be even a `type`.
A `function` is a `type` indeed.

Sure what does that mean? Well if `fix-point` of a function like `cos` is a `real-number`, and if you say that a `fix-point` can be a `function` itself,
that means the following:

```scala

There exists a `function`, whose fix-point can be a `funtion` itself. Hmmmmm.


```

## Stop explaining. Show an example.

A factorial of a number 3 is 3 * 2 * 1. It's implementation is as follows:

```scala

  val factorial: Int => Int =
    n => if (n == 0) 1 else n * factorial(n - 1)


```

Sure, that's easy. But let's say someone asked you to implement the same `factorial` such that the name of the function
shouldn't come in the body. Why? I think, answering this `Why` leads to applications that use `Fix Points`.

So, I request you to consider it as a challenge and a first step towards the understanding of `Fix` and `Recursion Schemes` and its wonderful application level usages
in various other talks/blogs.

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


```haskell



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


```scala

def fixPointOf(f: <someFn>): fixPointOf_SomeFn


```

This `fixPointOf` is also called "y combinator". The `Y` combinator is the one that converts a function to its fix-point.
However, before we call it as a combinator, let's call it as a function, because, to be a combinator it needs to satisfy a few conditions
that we will explain later. 



So, let's implement `y` such that it can take a function and returns its fix-point

```scala


// val factorial = almostFactorial(factorial)

// implies the following:

// If we are building a function called toFixPoint that takes `fn` as an argument then,
// that means the following

// toFixPoint (fn: Fn) = fn(toFixPoint(fn))
// Rename toFixPoint as Y, we will see later why
// Y(fn: Fn) = fn (Y(fn))

```

The above pseudo-code in Scala is


```scala

 // (A => A) => (A => A) is alligned to the signature of almostFactorial
  def Y[A](f: (A => A) => (A => A)) = f(Y(f))
  val factorial = Y(almostFactorial) // Stack overflow haha


```

Stack overflow is, when you call Y the following thing happens:


```scala

 f(Y(f)) ==
   f(f(f(f(f.......))))


```
In other words, it tries to compute the fix-point function for you and it never ends.

Lambda saves us from this stack overflow - in a surprising way.


```scala

  def Y[A](f: (A => A) => (A => A)): A => A = f(Y(f)) // Stack unsafe
  // same as
  def Y[A](f: (A => A) => (A => A)): A => A = f(a => Y(f)(a)) // Stack safe


```

Now my factorial implementation is

```scala

 val factorial = Y(almostFactorial)

 // factorial(3) is 6


```

Now, that works. However, we could further improve things.

The `Y` function that we defined (the function that takes another function and returns a fix-point function)
has a restricted shape.

```scala

  def Y[A](f: (A => A) => (A => A)): A => A = f(Y(f)) // Stack unsafe


```

`(A => A) => (A => A)` is equivalent to `A => A` in types. In other words.

The following will compile.

```scala

    def Y[A](f: A => A): A = f(Y(f)) // Stack unsafe
    val factorial = Y(almostFactorial)


```

However, it's sort of fairly hacky to make our latest `Y` stacksafe using our lambda technique.
Try yourself if you are having a doubt. 
This is sort of solveable though. However, for the time being, we can keep the same original `Y` as it is but with only a minor change.
We can make the return type a `B` instead of `A` to allow more flexiblity.

In the case of `factorial` which is `Int => Int`, `B` is `A` itself.


```scala

  def Y[A](f: (A => B) => (A => B)): A => B = f(a => Y(f)(a)) // Stack safe
  val factorial = Y(almostFactorial)


```

## Is our `y` a combinator - No!

To become a combinator, the implementation of `y` shouldn't have any free variables in it.

For example: The below lambda is a combinator, because you can replace `f` with its implementation
in all occurances of `f`.

```scala

val f = (x: Int) => x + 1


```

However, our `y` is not a combinator, the implementation `f(y(f))` has a free variable, and that's `y` itself.
An explicit recursion here made it not call as a combinator.

```scala

def y[A, B] = (f: (A => B) => (A => B)) => f(y(f))


```

Well let's revisit our factorial, and try abstracting out recursion again, 
and call our function `partFactorial`. You will see why we are not calling it as
`almostFactorial` later.

```scala
  def partFactorial(f: Int => Int): Int => Int = n => if (n == 0) 1 else n * f(n - 1)



```

Our real factorial is when `f` is partFactorial itself.

```scala

// Will not compile
// well partFactorial _ is of the type `Int => Int => Int => Int`.
// However, `partFactorial` method above accepts `Int => Int`
def factorial = partFactorial(partFactorial _) 


```

Ok, let's do a hack. We have Any in scala, and see how it goes.

```scala

def partFactorialv0(self: Any): Int => Int = 
  n => if (n == 0) 1 else n * self.asInstanceOf[Any => (Int => Int)](self)(n - 1)

def factorial(n: Int) = partFactorialv0(partFactorialv0 _)

// Works, factorial(3) == 6


```

Why did we do this? Well, let's extract out our `almostFactorial` discussed above from this implementation.

Repeating `almostFactorial` here:

```scala

def almostFactorial(f: Int => Int): Int => Int = 
  n => if (n == 0) 1 else n * f(n-1)


```


```scala

 def partFactorialv1(self: Any): Int => Int = {
    val f = self.asInstanceOf[Any => (Int => Int)](self) // val f = self(self) 
    n => if (n == 0) 1 else n * f(n - 1) // almost n factorial
  }

// Hmmm, almost there. But it could be written as:


def partFactorialv2(self: Any): Int => Int = {
    val f = self.asInstanceOf[Any => (Int => Int)](self) // val f = self(self) 
    n => if (n == 0) 1 else n * f(n - 1) // almost n factorial
  }

def partFactorialv3(self: Any): Int => Int = {
    val selfSelf = self.asInstanceOf[Any => (Int => Int)](self) // val f = self(self) 
    almostFactorial(selfSelf) // almost n factorial
  }

// stack safety issues come into picture here, hence factorialV3(4) will not work
def factorialV3 = partFactorialv3(partFactorialv3 _) 


```

Sure, we kind of extracted/reused almostFactorial in our latest implementations. 
You might be wondering why we are we doing this long loop of refactoring. 
I agree, but I request you to keep your patience and read on.

All our implementations of factorial is `partFactorial*(partFactorial* _)`.
Let's get rid of it and try having only one single function called `factorial`. That means
`partFactorial` implementation will become part of the `factorial` itself.


```scala

 // This was our partFactorial
 def partFactorialv4: Any => (Int => Int) = {
    (self: Any) => almostFactorial(self.asInstanceOf[Any => (Int => Int)](self)) // almostFactorial self self
  }

 // Moved inside our factorial implementation now.
 def factorialV4: Int => Int = {
    val partFactorial = 
       (self: Any) => almostFactorial(self.asInstanceOf[Any => (Int => Int)](self)) // almostFactorial self self

    partFactorial(partFactorial)   
  }  

// Well, lets rename this partFactorial to x
 def factorialV5: Int => Int = {
    val x = 
       (self: Any) => almostFactorial(self.asInstanceOf[Any => (Int => Int)](self)) // almostFactorial self self

    x(x)   
  }

// Let's apply our trick: Replace { val x: A = a; x(x) } with { ((x: A) => x(x))(a) }
 def factorialV6: Int => Int = {
    ((x: Any => (Int => Int)) => x(x))(
      (self: Any) => almostFactorial(self.asInstanceOf[Any => (Int => Int)](self))
    )
  }

// Just renaming self to x, and obviously it doesn't change the meaning of the program.
// At this stage, I really recommend you try this code compile in your IDE
// instead of reasoning it without a code, because languages are always complex in some way.
  def factorialV7: Int => Int = {
    ((x: Any => (Int => Int)) => x(x))(
      (x: Any) => almostFactorial(x.asInstanceOf[Any => (Int => Int)](x))
    )
  }



```

As functional programmers, we cannot allow `almostFactorial` to be part of the implementation.
Let's abstract that out.

```scala

// It's no more factorial because we extracted out almostFactorial as `f`.
def makeRecursive(f: (Int => Int) => (Int => Int)) = {
        ((x: Any => (Int => Int)) => x(x))(
      (x: Any) => f(x.asInstanceOf[Any => (Int => Int)](x))
    )
  }


def factorialV8 = makeRecursive(almostFactorial)
// Works? stack overflow again. But we will fix it later. Hold tight.



```

Before we fix the stack overflow error, let's simply try to prove that 
makeRecursive is our `y`(implemented above). 

And why do we need to prove ? `makeRecursive` is our real `y` that is devoid of free variables.
In other words, we are going to prove that there is a `combinator` version of `def y(f) = f(y(f))`


```scala

// makeRecursive is renamed to yCombinator0
def yCombinator0(f: (Int => Int) => (Int => Int)) = {
  ((x: Any => (Int => Int)) => x(x))(
    (x: Any) => f(x.asInstanceOf[Any => (Int => Int)](x))
  )
}

def factorialV9 = yCombinator0(almostFactorial)

// Below code summarises the following:
// y f = (lambdaXfxx)((lambdaXfxx))
// where lambdaXfxx =  (x: Any) => f(x.asInstanceOf[Any => (Int => Int)](x))
def yCombinator1 = 
    (f: (Int => Int) => (Int => Int)) => {
       (
         (x: Any) => f(x.asInstanceOf[Any => (Int => Int)](x))
        )(
         (x: Any) => f(x.asInstanceOf[Any => (Int => Int)](x))
        )
  }

 // The final refactoring
 def yCombinator2 = 
    (f: (Int => Int) => (Int => Int)) => {
      val lambdaXFxx = ((x: Any) => f(x.asInstanceOf[Any => (Int => Int)](x)))
      f(lambdaXFxx(lambdaXFxx))
  }



```

There lies the proof. Can you see that?

```scala

// `yCombinator1` says the following:

y f = (lambdaXfxx)((lambdaXfxx))


// `yCombinator2` says the following

y f = f ((lambdaXfxx)((lambdaXfxx)))
y f = f (y f)

// Hence, we proved our `def yCombinator2(f) = <expr>` is same as `def y(f) = f(y(f))`


```

In summary, `yCombinator2` is our real `y` combinator

## Remove stack overflow from y combinator

May be we can apply the lambda technique we used before 
i.e, 

```scala

{ val x: A => A = f(z) } is same as {val x : A => A = a => f(z)(a)}


```

All thats done here is, apply an extra lambbda, wherever possible.

```scala

// Stack safe
def yCombinator = 
    (f: (Int => Int) => (Int => Int)) => {
      val lambdaXFxx = ((x: Any) => f((y: Int) => x.asInstanceOf[Any => (Int => Int)](x)(y)))
      f(lambdaXFxx(lambdaXFxx))
  }

def factorial = yCombinator(almostFactorial) // Works !!


```

## Generalise y combinator

```scala

 def y[A, B] = 
  (f: (A => B) => (A => B)) => {
    val lambdaXFxx = ((x: Any) => f((y: A) => x.asInstanceOf[Any => (A => B)](x)(y)))
    f(lambdaXFxx(lambdaXFxx))
  }

// In scala3, you can also write something like this if you want. 
// It removes `Any`, but it doesn't really matter :D

def yScala3_v0[A, B] = 
  (f: (A => B) => (A => B)) => {
    val lambdaXFxx:[Z] => Z => (A => B) = ([Z] => (z: Z) => f((y: A) => z.asInstanceOf[Z => (A => B)](z)(y)))
    f(lambdaXFxx(lambdaXFxx))
  }

// Well, why do we need an asInstanceOf. If you think about it, the above code is equivalent to

def yScala3[A, B] = 
  (f: (A => B) => (A => B)) => {
    lazy val lambda:[Z] => Z => (A => B) = ([Z] => (z: Z) => f((y: A) => lambda(z)(y)))
    f(lambda(lambda))
  }

// I think I liked that one
def y[A, B] = 
  (f: (A => B) => (A => B)) => {
    lazy val lambda:[Z] => Z => (A => B) = 
      ([Z] => (z: Z) => f((y: A) => lambda(z)(y)))
    f(lambda(lambda))
  }
  
def factorial = y(almostFactorial)


```

Ah! That was a long loop. I know, but not only you learned fix-points, you learned its 
implementation in scala through `y f = f (y f)` and proved that there exists a combinator, that
is devoid of free variables that can achieve the same result in both strict and lazy languages.

Thanks, and Well done for reading such a long blog. 
If you are interested to read explanations in `Lisp` or `Scheme` language,
go on and read this: https://mvanier.livejournal.com/2897.html
