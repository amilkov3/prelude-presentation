
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

### Referential Transparency

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

Where Haskell requires you perform all effects in `IO`. So we need a referentially transparent `IO` monad
(unlike `Future` for example) for Scala

#### Enter `cats-effect`

---?image=assets/cats-effect.png&position=center&size=auto 80%

---

```scala
val res: IO[Array[Byte]] = IO(httpClient.get("https://en.wikipedia.org/wiki/Side_effect_(computer_science)"))
```

---

We can take this further by programming against a typeclass API instead of a concrete impl

```scala
def getArticle[F[_]](implicit ev: Sync[F]): F[Array[Byte]] =
  ev.delay(httpClient.get("https://en.wikipedia.org/wiki/Side_effect_(computer_science)"))

getArticle[IO].unsafePerformSync()

(IO.shift(ec) *> getArticle[IO]).runAsync{
  case Right(_) => IO.unit
  case Left(e) => IO(logger.error(s"GET article failed with: ${e.getMessage}"))
}.unsafeRunSync()

```

@[1-3](`Sync` typeclass has a lazy `delay` method to suspend an effect)
@[4](Perform effect)
@[6-9](`Async` typeclass has `runAsync` method for composing an RT callback)

---

### Utils

This pattern will serve as the foundation for:
* HTTP wrapper
* Cache wrapper

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

  override def get[A: Decoder](
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

  override def post[A: Encoder](
    url: Url,
    body: A,
    headers: Map[String, String] = Map.empty
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

### Example

```scala
@JsonCodec(decodeOnly = true)
case class Post(
  userId: Int,
  id: Int,
  title: String,
  body: String
)

implicit lazy val sttpBackend: SttpBackend[IO, Nothing] =
  new IOHttpURLConnectionBackend

class IOHttpURLConnectionBackend extends SttpBackend[IO, Nothing] {

  val client = HttpURLConnectionBackend()

  override def send[T](request: Request[T, Nothing]): IO[Response[T]] =
    IO(client.send(request))

  override def close(): Unit = ()

  /** Translates `cats.MonadError` to `com.softwaremill.sttp.MonadError[IO]` */
  override def responseMonad: com.softwaremill.sttp.MonadError[IO] = new com.softwaremill.sttp.MonadError[IO] {
    override def error[T](t: Throwable): IO[T] = IO.raiseError(t)
    override def flatMap[T, T2](fa: IO[T])(f: T => IO[T2]): IO[T2] = fa.flatMap(f)
    override def unit[T](t: T): IO[T] = IO(t)
    override def map[T, T2](fa: IO[T])(f: T => T2): IO[T2] = fa.map(f)
    override def handleWrappedError[T](rt: IO[T])(h: PartialFunction[Throwable, IO[T]]): IO[T] = rt.recoverWith(h)
  }
}

val client = new JsonHttpClient[IO](new HttpClientConf {
  override val readTimeout: FiniteDuration = 2.seconds
})

val url = Url(
  Host.unsafeCreate("jsonplaceholder.typicode.com"),
  Path.unsafeCreate("/posts/1"),
  true
)

val res: HttpResponse[Either[JsonErr, Post]] =
  client.get[Post](url).unsafeRunSync()
```

@[1-29](sttp backend impl and our type)
@[30-42](our call)

---

### Cache typeclasses

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

### Extraneous additional util features

* App Error Hierarchy
* `prelude-mongo`: functional wrapper around Mongo Casbah driver

#### Not discussed
* `prelude-geo`: functional wrapper around `jgeohash` Java geo library

---

### App Error Hierarchy

```
                                             AppFailure
                                           /     |       \
                                         /       |        \
                                        /        |         \
                              UserFailure InternalFailure UpstreamFailure
```

---

### `AppFailure`

```scala
/** For raising errors inside our effect monad */
final case class AppException(e: AppFailure) extends Exception(e.message, e.cause.orNull)

/** Top level failure type all concrete UpsFailures inherit from */
trait AppFailure {
  def message: String
  def cause: Option[Throwable]
}
```

---

### `InternalFailure`

```scala
final case class InternalFailure private(
  desc: String,
  component: InternalComponent,
  cause: Option[Throwable] = None
)  extends  AppFailure {

  override val message: String = s"${component.toString} failed with: $desc"
}

object InternalFailure {
  def apply(message: String, component: InternalComponent) = {
    new InternalFailure(message, component)
  }
}

trait InternalComponent
case object Encryption extends InternalComponent
case object Decryption extends InternalComponent
case object ThreadPoolExhausted extends InternalComponent
```

@[1-14](What the type looks like)
@[16-19](Your components)

---

### `UpstreamFailure`

```scala
/** Represents a failure in some upstream component like a db or service */
final case class UpstreamFailure private (
  component: UpstreamComponent,
  cause: Option[Throwable] = None
) extends AppFailure {

  override val message: String = s"Upstream failure. Info: ${component.message}"
}

object UpstreamFailure {
  def apply(component: UpstreamComponent): UpstreamFailure = {
    new UpstreamFailure(component)
  }
}

/** Extend this to add upstream components that may fail, like db ops for example */
trait UpstreamComponent {
  def code: Int
  def message: String
}

trait ServiceFailure extends UpstreamComponent {
  def name: String
  override val code = 503
}

/** Service returned a success, but payload could not be decoded */
case class ServiceInvalidPayload (
  name: String,
  payload: InvalidPayload
) extends ServiceFailure {
  override val message: String = s"Invalid payload from $name. Info: ${payload.message}"
}

/** Service returned an error */
case class ServiceHttpError (
  name: String,
  respCode: Int,
  body: String
) extends ServiceFailure {
  override val message: String = s"$name call returned a $respCode. Body: $body"
}

/** Http client couldn't even reach the service */
case class ServiceUnreachable (
  name: String,
  url: Url,
  ex: Throwable
) extends ServiceFailure {
  override val message: String =
    s"Error while contacting service: $name. URL: ${url.show}. Info: ${ex.getMessage}"
}
```

@[1-14](Your type)
@[16-52](Component example)

---

##### A more nuanced example of programming in terms of the `cats-effect` typeclass API: Mongo driver wrapper

---

```scala
final class MongoCollectionWrapper(repr: MongoCollection) {

  repr.setReadPreference(Secondary.underlying)

  def insertOneF[A <: Product, F[_] : Effect](
      a: A,
      checkUniquenessWith: MongoDBObject = DBObject.empty
  )(implicit
    ev: BsonCodec.Aux[A, BsonDocument],
    tt: TypeTag[A]
  ): F[Unit] = {
    checkUniquenessWith.isEmpty.fold(
      ().pure[F],
      Effect[F].delay(repr.findOne(checkUniquenessWith)).flatMap(_.empty.fold(
        ().pure[F],
        ApplicativeError[F, Throwable].raiseError[Unit](
          new AppException(
            UpstreamFailure(DuplicateRecord(a))
          )
        )
      ))
    ).attempt // F[Either[Throwable, Unit]]
      .flatMap(either =>
        ApplicativeError[F, Throwable].fromEither[WriteResult](
          either.map(_ => repr.insert(new BasicDBObject(ev.encode(a))))
        )
      ) // F[WriteResult]
      .handleErrorWith {
        case dupErr: AppException =>
          ApplicativeError[F, Throwable].raiseError[WriteResult](dupErr)
        case ex =>
          ApplicativeError[F, Throwable].raiseError[WriteResult](
            new AppException(
              UpstreamFailure(WriteFailure(ex.getMessage))
            )
          )
      }.flatMap {
      _.wasAcknowledged().fold(
        ().pure[F],
        ApplicativeError[F, Throwable].raiseError[Unit](
          new AppException(
            UpstreamFailure(WriteFailure(s"${tt.tpe} write was not acknowledged by Mongo"))
          )
        )
      )
    }
  }

  def upsertOneF[A <: Product, F[_] : Effect](
    a: A,
    queryDbO: DBObject,
    upsert: Boolean = true
  )(implicit
    ev: BsonCodec.Aux[A, BsonDocument],
    tt: TypeTag[A]
  ): F[Boolean] = {
    Effect[F].delay[F](repr.update(queryDbO, new BasicDBObject(ev.encode(a)), upsert))
      .handleErrorWith {
        case _: DuplicateKeyException =>
          ApplicativeError[F, Throwable].raiseError[WriteResult](
            new AppException(
              UpstreamFailure(DuplicateRecord(a))
            )
          )
        case ex =>
          ApplicativeError[F, Throwable].raiseError[WriteResult](
            new AppException(
              UpstreamFailure(WriteFailure(ex.getMessage))
            )
          )
      }.flatMap {
        res => res.wasAcknowledged().fold(
          (res.getN() == 1).fold(
            (!res.isUpdateOfExisting()).pure[F],
            ApplicativeError[F, Throwable].raiseError[Boolean](
              new AppException(
                UpstreamFailure(RecNotFound(
                  s"${upsert.fold("upsert failed", "update failed. Record likely not found.")}. Record queried with: ${queryDbO.asString}"
                ))
              )
            )
          ),
          ApplicativeError[F, Throwable].raiseError[Boolean](
            new AppException(
              UpstreamFailure(WriteFailure(s"${tt.tpe} write was not acknowledged by Mongo"))
            )
          )
        )
      }
  }

  def deleteOneF[F[_]: Effect](queryDbO: DBObject): F[Unit] = {
    Effect[F].delay[F](repr.findAndRemove(queryDbO))
      .flatMap(_.isDefined.fold(
        ().pure[F],
        ApplicativeError[F, Throwable].raiseError[Unit](
          throw new AppException(
            UpstreamFailure(RecNotFound(queryDbO.asString))
          )
        )
      ))
  }

  def deleteManyF[F[_]: Effect](queryDbO: DBObject): F[Unit] = {
    Effect[F].delay[F](repr.remove(queryDbO))
      .flatMap(_.wasAcknowledged().fold(
        ().pure[F],
        ApplicativeError[F, Throwable].raiseError[Unit](
          throw new AppException(
            UpstreamFailure(
              WriteFailure(s"${queryDbO.asString} deleteMany write was not acknowledged")
            )
          )
        )
      ))
  }
}

```

---

### Done!

* https://github.com/amilkov3/prelude-utils
* Gitter: amilkov1
* Email: amilkov3@gmail.com

