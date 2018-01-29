
# prelude utils

---

## Motivation

* Basically (well obviously to you guys) I'm trying to write Haskell in Scala because Haskell gets a lot of things right:
1. All effects must be performed inside of the `IO` monad, thereby forcing the developer to handle/reason about them appropriately
2. True RT (referential transparity).
3. True FP. i.e. can't write imperative code thereby stressing composibility

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

---

### 2 Referential transparity

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
      ApplicativeError[F, Throwable].raiseError[WriteResult](i
        new UpsException(
          UpstreamFailure(DuplicateRecord(caseClassInst))
        )
      DuplicateRecord)
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

### 3 No imperative muddling

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
  putStrLn (res)
```

obviously the equivalent in scala would akin to:

```scala
for {
  x <- Some(5)
  y <- Some(x + 3)
} yield y
```

Anyway the benefit to modeling imperativity like this is that you have think of and compose things
inside of an appropriate monadic context (for effects that would be `IO`, for things that may throw excpetions `Either`,
for values that may or may not be there `Option` (or `Maybe`) which generally leads to again increased composobility and
more explicit semantics behind whatever computation you are performing

---


No shit like `Future(5)` which is run immediately, producing some evaluted (eventually expression). i.e. you have no control over when the expression is actually run. Compare that to
