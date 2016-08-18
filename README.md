# finch
________



### finchとは

Twitterが開発したOSSのRPCシステム[finagle](https://github.com/twitter/finagle)の関数型ラッパー



build.sbt

```scala
val baseSettings = Seq(
  libraryDependencies ++= Seq(
    "com.chuusai" %% "shapeless" % shapelessVersion,
    "org.typelevel" %% "cats-core" % catsVersion,
    "com.twitter" %% "finagle-http" % finagleVersion,
    "org.scala-lang" % "scala-reflect" % scalaVersion.value,
    compilerPlugin("org.scalamacros" % "paradise" % "2.1.0" cross CrossVersion.full)
  )
```
  >"com.chuusai" %% "shapeless" % shapelessVersion,
  
  >"org.typelevel" %% "cats-core" % catsVersion,
  
  


oh shapeless and cats ...
  


finagleでは
```scala
abstract class Service[-Req, +Rep] extends (Req => Future[Rep])
```
というReq => Future[Rep]型の関数を実装してサーバーに渡す。


finchはこのServiceの実装を助けてくれる。



## hello world

```scala
import io.finch._
import com.twitter.finagle.Http

val api: Endpoint[String] = get("hello") { Ok("Hello, World!") }

Http.serve(":8080", api.toService)
```


Endpointという関数をtoServiceで変換するとfinagleで受け取れる(Service[Request, Response]になる)ようになる。
Endpointを組み合わせてapiを作っていく。




## Endpoint

Endpointは
```scala
Request => Option[Future[Output[A]]]
```
の型の関数


* Optionはrequestがパスにマッチしているかどうかを表していてNoneなら404
* Futureは非同期の計算で実行時例外が起きると500が帰る
  * scala.util.Futureではなくtwitter.util.Futureなので注意
* Aがレスポンスの内容 
* ビジネスロジックはEndpoint[A]のAを作成することになる。


実際に使ってみる

```scala
get("hoge" :: "fuge") // => /hoge/fuge のパスにマッチ
get("hello"::param("name")) // =>/hello?name=... にマッチ
get("hello"::paramOption("name")) //=> ?nameがなくてもマッチ 
get("hello"::params("name")) //=> hello?name=...&name=...に複数のnameにマッチ
get("hello"::paramsNonEmpty("name")) //=> /hello　nameが一つもないとエラー
```


param("fuge").as[A]とすると fuge=Aの型だけマッチし型がマッチしないとエラーになる。


finchがデフォルトではサポートしていない型をデコードしたい場合は
```scala
implicit val dateTimeDecoder: DecodeRequest[DateTime] =
  DecodeRequest.instance(s => Try(new DateTime(s.toLong)))
```
という `String => Try[A]`の関数を実装すれば良い。　


デフォルトでパスにマッチしてれる型は
string(), boolean(), uuid(), int(), long()などがある。
```scala
get("helo"::string("name")::long("age")) // hello/[String]/[Long]だけにマッチ
```



## EndPointの合成


あるパスにマッチかつ続けて別のパスにマッチさせたい場合は

`::`で合成していく


`::`で合成するとshapelessのHListになる
a :: b の場合は a にマッチかつ bにマッチするパターン
```scala
val user: Endpoint[String::Long::HNil] = string("name")::long("age")
```


あるパスにマッチしなかったら別のパスにマッチさせたい場合は
`:+:`で合成していく


`:+:`で合成するとshapelessのCoproduct(直和)になる



例えばCRUDアプリを作る時などは

```scala
val get = get("user"::long){p: Long => Ok(...)}
val put = put("user"::long){p: Long => Ok(...)}
val del = delete("user"::long){p: Long => Ok(...)}
val api = get :+: put :+: del
```

こういう風に組み合わせる


shapeless使っているので
case classの変換も

クエリパラメーターからUser型に変換
```scala
case class User(name: String, age: Int)
val api: Endpoint[String] = get("hello" :: Endpoint.derive[User].fromParams) { u: User => 
  Ok(u.toString) 
}
```


bodyからUser型に変換
```scala
case class User(name: String, age: Int)
val api: Endpoint[String] = get("hello" :: body.as[User) { u: User => Ok(u.toString) }
```




#validation
val rule = ValidationRule[A](message: String)(f: A => Boolean)
を定義する

```scala
val bePositive = ValidationRule[Int]("be postive" )(_ > 0)

  val user = (
      param("name") ::
      param("age").as[Int].should(bePositive)
    ).as[User]
```


合成も簡単にできる
```scala
val bePositive = ValidationRule[Int]("be postive" )(_ > 0)
val under18 = ValidationRule[Int]("be postive" )(_ < 18)

  val user = (
      param("name") ::
      param("age").as[Int].should(bePositive).should(under18)
    ).as[User]
```


shouldがEndPointを返すおかげで型エラーに悩まされないで合成できる
playのformよりずっといい



##まとめ
* finchの関数がほとんどがEndpoint型を返すのでとにかく合成しやすい
  * Endpointを合成するとEndpoint
  * Endpointクラスが持つpublic methodの殆どがEndpoingを返す。
* sprayみたいに書けるがコードの実装がsprayに比べだいぶシンプルなのでより読みやすいと思う。







