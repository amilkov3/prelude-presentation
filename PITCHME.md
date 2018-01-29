
# prelude utils

---

## Motivation

Basically (well obviously to you guys) I'm trying to write Haskell in Scala because Haskell gets a lot of things right:

1. _Effect Abstraction_
2. _Referential Transparency_
3. _No imperative muddling_

---

### 1. Effect Abstraction


We've all seen code like this which is perfectly legal regretably:

```scala
// perform IO willy nilly
val res = mongoColl.find(query)
```

What is better is:

```scala
Either.catchNonFatal(mongoColl.find(query))
```

But this is cheating/not giving you the full picture:

```scala
trait Sync[F[_]] {
  def sync[A](f: F[A]): F[A]
}

def find[F: Sync](q: DBOBject) = sync[F](mongoUserColl.find(q))

```

All effects must be performed inside of an effect typeclass API (in Haskell the defacto instance of this typeclass (called `MonadIO` is the `IO` monad. Now Scala has `IO` and effectful typeclasses too. This forces the developer to handle/reason about effects appropriately

---

### 2. Referential transparency

```scala
val ref: F[Either[Throwable, WriteResult]] = find(someQuery).attempt

// we don't lose RT until we actually perform the effect at the end of the world
ref.unsafePerformIO: Either[Throwable, WriteResult]

// for clarity, basically the same as:
Either.catchNonFatal(find(someQuery).unsafePerformIO): Either[Throwable, WriteResult]

```

Lets do something a bit more involved:

```scala

// say we have a unique index and we want to catch `com.mongodb.DuplicateKeyException` exceptions
sync[F](mongo.insert(caseClassInst.toDBObject))
  .handleErrorWith {
    case dupErr: com.mongodb.DuplicateKeyException =>
      ApplicativeError[F, Throwable].raiseError[WriteResult](
        new UpsException(
          UpstreamFailure(DuplicateRecord(caseClassInst))
        )
      )
    case ex =>
      ApplicativeError[F, Throwable].raiseError[WriteResult](
        new UpsException(
          UpstreamFailure(WriteFailure(ex.getMessage))
        )
      )
  }

```

So you see how we can very neatly compose an expression tree, an in memory representation of our program
that can be executed when we so choose, making it easier to reason about what we're writing and increasing
testability

---

### 3. No imperative muddling

We've all seen shit like this:

```scala
def spareMe = {
  val a = performSomeSideEffectReturningA
  val b = decodeABFromANotCatchingExceptions(a)
  b * 3 + 2 // oh god
}
```

This is Java, not Scala

In Haskell the only way to simulate imperative programming is via monadic composition

```haskell
main :: IO ()
main = do
  line <- getLine
  let res = "you said: " ++ line
  putStrLn res
```

Obviously the equivalent in Scala would akin to:

```scala
for {
  x <- Some(5)
  y <- Some(x + 3)
} yield y
```

Anyway the benefit to modeling imperativity like this is that you have think of and compose things
inside of an appropriate monadic context (for effects that would be `IO`, for things that may throw exceptions `Either`,
for values that may or may not be there `Option` (or `Maybe`)) which generally leads to, again, increased composability and
more explicit semantics behind whatever computation you are performing

---


No shit like `Future(5)` which is run immediately, producing some evaluted (eventually expression). i.e. you have no control over when the expression is actually run. Compare that to
