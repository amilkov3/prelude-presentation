
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
that can be executed when we so choose, via `unsafePerformIO()` or async via `unsafePerformAsync(cb)` making it easier to reason about what we're writing and increasing
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

Obviously the equivalent in Scala would be akin to:

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

# Utils

Piggy backing off of effect abstraction:
* Effect typeclasses

Which will serve as the foundation for:
* HTTP wrapper
* Cache wrapper

And then some unrelated useful components:
* Error hierarchy
* Json
* Geo

---

### HTTP

Here's what the interface looks like:

```scala
trait EfJsonHttpClient[F[_]] extends {

  // `JsonDecodable` just points to typeclass `io.circe.Decoder[A]`
  def get[A: JsonDecodable](url: Url): F[HttpResponse[Either[JsonErr, A]]]

  // same with `JsonEncodable`
  def post[A: JsonEncodable](
    url: Url,
    body: A,
    headers: Map[String, String] = Map.empty
  ): F[HttpResponse[Unit]]
}


```

This is a simple wrapper around the `com.softwaremill.sttp` library, which already employs a similar effect abstraction pattern
and allows you to wrap whatever underlying client you so choose


---

### And our impl

```scala
final class JsonHttpClient[F[_]: Effect](
  conf: HttpConfig
  )(implicit ev: SttpBackend[F, Nothing])
  extends EfJsonHttpClient[F]{

  private val baseC = sttp.readTimeout(conf.connectionTimeout)

  def respAsJson[B](implicit ev: JsonDecodable[B]): ResponseAs[Either[JsonErr, B], Nothing] = {
    asString.map{s =>
      for {
        json <- Json.parse(s).leftMap(_.asInstanceOf[JsonErr])
        b <- ev.decodeJson(json.repr).leftMap(err => (new JsonDecodeErr(err.getMessage())): JsonErr)
      } yield b
    }
  }

  override def get[A: JsonDecodable](url: Url): F[HttpResponse[Either[JsonErr, A]]] = {
    baseC
      .get(SUrl(
        url.scheme.asString,
        None,
        url.host.repr,
        None,
        url.path.toList,
        List.empty[SUrl.QueryFragment],
        None
      ))
      .response(respAsJson[A])
      .send[F]()
      .map(new HttpResponse(_))
  }

  override def post[A: JsonEncodable](
    url: Url,
    body: A,
    headers: Map[String, String]
  ): F[HttpResponse[Unit]] = {
    baseC
      .post(Url(
        url.scheme.asString,
        None,
        url.host.repr,
        url.port,
        url.path.toList,
        List.empty[Url.QueryFragment],
        None
      ))
      .body(body.toJson.spaces2, StandardCharsets.UTF_8.name)
      .send[F]()
      .map(new HttpResponse(_).mapBody(_ => ()))
  }
}
```

