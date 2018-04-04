
# prelude utils

---

## Motivation

`Functional programming + static typing = Haskell`

`... Functional programming + static typing ... = Scala`

`Scala --> Haskell`

#### Primer

- Referential Transparency |
- Composition over Imperativity |

#### Meat

- Effect Abstraction |

<!--  hello -->

---

### Referential Transparity

_An expression can be replaced by its value (or anything with the same value) without changing
the results of the program_

```scala
val x = println("non RT")
f(x, x)

def x = println("RT")
f(x, x)

val x = Future(println("non-RT"))
```

@[4-5](But `println` is still non RT  where the equivalent in Haskell, `putStrLn`, is due to lazyness)
@[7](The fundamental concurrency primitive is non RT)

#### Benefits:

- Reasonability |
- Testability |
- Express programs as in memory expression trees |

[//]: # (hello)

---

### Composition over Imperitivity

Code like this evince bad memories?

```scala
def pleaseFixMe(something: Int) = {
  val a = performSomeSideEffectReturningA
  val b = decodeABFromANotCatchingExceptions(a)
  b.setFoo(new Foo(something)) // oh god
  b
}
```

[//]: # (You're assigning arbitrary expressions, which may or may not occur in a context, monadic or not, to vals, which then may or may not be used but may depend on a previous imperative declaration, and you're mutating a val)

---

Haskell forbids such an imperative style but can be mimiced with:

```haskell
test :: Int
test = let x = 2
           y = x + 2
       in x + y + z + a
       where z = 2
             a = z + 2

readIn :: IO ()
readIn = do
  line <- getLine
  let res = "you said: " ++ line
  putStrLn res

```
@[2-4](let clause)
@[5-6](where clause)
@[8-12](monadic composition)

---

```scala
for {
  x <- Some(5)
  y <- Some(x + 3)
} yield y
```

@[1-4](That last Haskell example should look familiar to)

#### Benefits

* Reasonability |
* Testability |
* Monadic context |

[//]: # (Testability -- via composability since you can test individual functions or chained functions)

[//]: # (Anyway the benefit to modeling imperativity like this is that you have think of and compose things inside of an appropriate monadic context (for effects that would be `IO`, for things that may throw exceptions `Either`, for values that may or may not be there `Option` (or `Maybe`)

---

### So how can we apply composition and RT to side-effects?

---

### IO

The trouble with Scala

```scala
val res: Array[Byte] = httpClient.get("https://en.wikipedia.org/wiki/Side_effect_(computer_science)")
```

@[1](This is perfectly legal)

Where Haskell requires you perform all effects in `IO`. So we need an `IO` monad for Scala that
has referential transparity as a property unlike `Future` for example

#### Enter `cats-effect`

---?image=assets/cats-effect.png&position=center&size=auto 80%

---

```scala
val res: IO[Array[Byte]] = IO(httpClient.get("https://en.wikipedia.org/wiki/Side_effect_(computer_science)"))
```

We can take this further by programming against a typeclass API instead of a concrete impl

```scala
def getArticle[F[_]](implicit ev: Sync[F]): F[Array[Byte]] =
  ev.delay(httpClient.get("https://en.wikipedia.org/wiki/Side_effect_(computer_science)"))
```

@[2-3](`Sync` typeclass has a lazy `delay` method to suspend an effect)

---

### Utils

This pattern will serve as the foundation for:
* HTTP wrapper
* Cache wrapper

And then some unrelated useful components:
* _Error hierarchy_
* _Json_
* _Geo_

---

### HTTP wrapper

Lets start with our interface

```scala
abstract class EfJsonHttpClient[F[_]: Effect] {

  def get[A: Decoder](
    url: Url,
    headers: Map[String, String] = Map.empty
  ): F[HttpResponse[Either[JsonErr, A]]]

  def post[A: Encoder](
    url: Url,
    body: A,
    headers: Map[String, String] = Map.empty
  ): F[HttpResponse[Unit]]
}
```

---

#### Wrap `com.softwaremill.sttp`

```scala
class JsonHttpClient[F[_]: Effect](
  conf: HttpClientConf
)(implicit ev: SttpBackend[F, Nothing]) extends EfJsonHttpClient[F]{

  private val client = sttp.sttp.readTimeout(conf.readTimeout)

  override def get[A: JsonDecodable](
    url: Url,
    headers: Map[String, String] = Map.empty
  ): F[HttpResponse[Either[JsonErr, A]]] = {
    client
      .get(SUrl(
        url.isHttps.fold("https", "http"),
        None,
        url.host.repr,
        None,
        url.path.toList,
        List.empty[SUrl.QueryFragment],
        None
      ))
      .headers(headers)
      .response(sttp.asString.map{s =>
        for {
          json <- Json.parse(s).leftMap(err => JsonParseErr(err.message): JsonErr)
          a <- json.as[A].leftMap(err => JsonDecodeErr(err.message): JsonErr)
        } yield a
      })
      .send[F]()
  }

  override def post[A: JsonEncodable](
    url: Url,
    body: A,
    headers: Map[String, String]
  ): F[HttpResponse[Unit]] = {
    client
      .post(SUrl(
        url.isHttps.fold("https", "http"),
        None,
        url.host.repr,
        url.port,
        url.path.toList,
        List.empty[SUrl.QueryFragment],
        None
      ))
      .headers(headers)
      .body(body.asJson.spaces2, StandardCharsets.UTF_8.name)
      .send[F]()
      .map(r => r.copy(body = r.body.map(_ => ())))
  }
}
```

@[5-29](`get`)
@[29-50](`post`)

---

### Cache typeclass

```scala
trait SerializableKV[In] {
  type Out
  def serialize(k: In): Out
}

object SerializableKV {
  /** The `Aux` pattern creates a dependent type association between
    * , in this case, an A and a B */
  type Aux[A, B] = SerializableKV[A] { type Out = B }

  def apply[A, B](implicit ev: Aux[A, B]): Aux[A, B] = ev

  def instance[A, B](f: A => B): Aux[A, B] = new SerializableKV[A] {
    type Out = B
    override def serialize(k: A): B = f(k)
  }
}

trait DeserializableV[Out] {
  type In
  def deserialize(k: Out): Either[AppFailure, In]
}

object DeserializableV {
  type Aux[A, B] = DeserializableV[A] { type In = B }

  def apply[A, B](implicit ev: Aux[A, B]): Aux[A, B] = ev

  def instance[A, B](f: A => Either[AppFailure, B]): Aux[A, B] = new DeserializableV[A] {
    type In = B
    override def deserialize(k: A): Either[AppFailure, B] = f(k)
  }
}

trait CacheableKVPair[K] {
  type V
}

object CacheableKVPair {
  type Aux[K, VV] = CacheableKVPair[K] { type V = VV }

  def apply[K, V](implicit ev: Aux[K, V]): Aux[K, V] = ev

  def instance[K, VV]: Aux[K, VV] = new CacheableKVPair[K] {
    type V = VV
  }
}
```

@[1-17](Typeclass for serializing a cache key or value `In` to the underlying cache's key type `Out`)
@[18-33](Typeclass for deserialzing the underlying cache value representation `Out` to a type `In`)
@[35-47](Typeclass used simply to create a compilation constraint ensuring that a key value pair `K` `V` may be cacheable)

---

### Cache wrapper

```scala
trait EfCacheClient[F[_], KK, VV] {
  def get[K, V](k: K)(implicit
    ev1: CacheableKVPair.Aux[K, V],
    ev2: SerializableKV.Aux[K, KK],
    ev3: DeserializableV.Aux[VV, V]
  ): F[Option[Either[AppFailure, V]]]

  def put[K, V](k: K, v: V)(implicit
    ev1: CacheableKVPair.Aux[K, V],
    ev2: SerializableKV.Aux[K, KK],
    ev3: SerializableKV.Aux[V, VV]
  ): F[Unit]
}

abstract class EfBaseCacheClient[F[_]: Effect, KK, VV] extends EfCacheClient[F, KK, VV] {

  override final def get[K, V](k: K)(implicit
    ev1: CacheableKVPair.Aux[K, V],
    ev2: SerializableKV.Aux[K, KK],
    ev3: DeserializableV.Aux[VV, V]
  ): F[Option[Either[AppFailure, V]]] = {
    Effect[F].pure(
      get(ev2.serialize(k)).map(ev3.deserialize)
    )
  }

  override final def put[K, V](k: K, v: V)(implicit
    ev1: CacheableKVPair.Aux[K, V],
    ev2: SerializableKV.Aux[K, KK],
    ev3: SerializableKV.Aux[V, VV]
  ): F[Unit] = {
    Effect[F].pure(
      put(ev2.serialize(k), ev3.serialize(v))
    )
  }

  protected def get(k: KK): Option[VV]

  protected def put(k: KK, v: VV): Unit
}
```

@[1-13](Interface for generic client)
@[15-40](Base impl)

---

### Example

```scala
case class Foo(x: String, y: Int)
case class Bar(a: Double, b: Boolean)

implicit val serializableKVFoo: SerializableKV.Aux[Foo, String] =
  SerializableKV.instance[Foo, String](foo => s"${foo.x},${foo.y}")

implicit val serializableKVBar: SerializableKV.Aux[Bar, Array[Byte]] =
  SerializableKV.instance[Bar, Array[Byte]](
    Codec[Bar].encode(_).toEither.asRight.toByteArray
  )

case object DeserializationFailure extends InternalComponent

implicit val deserializableV: DeserializableV.Aux[Array[Byte], Bar] =
  DeserializableV.instance[Array[Byte], Bar](
    ba => Codec[Bar].decode(BitVector(ba))
      .toEither
      .leftMap(err => InternalFailure(err.message, DeserializationFailure))
      .map(_.value)
  )

implicit val cacheableKVPair: CacheableKVPair.Aux[Foo, Bar] = CacheableKVPair.instance

final class MemCacheClient[F[_]: Effect, KK, VV] extends EfBaseCacheClient[F, KK, VV] {

  val cache  = scala.collection.mutable.Map.empty[KK, VV]

  override protected def get(k: KK): Option[VV] = cache.get(k)

  override protected def put(k: KK, v: VV): Unit = cache.put(k, v)
}

val cacheClient = new MemCacheClient[Id, String, Array[Byte]]

property("should put and get, serializing and deserializing correctly") {
  val foo = Foo("hello", 2)
  cacheClient.put[Foo, Bar](foo, Bar(1.5d, true))
  cacheClient.get[Foo, Bar](foo).asSome.asRight
}
```

---

### Effect Abstraction

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

