Slick-pg
========

[![Join the chat at https://gitter.im/tminglei/slick-pg](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/tminglei/slick-pg?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

[Slick](https://github.com/slick/slick "Slick") extensions for PostgreSQL, to support a series of pg data types and related operators/functions.

#### Currently supported pg types:
- ARRAY
- Date/Time
- Enum
- Range
- Hstore
- LTree
- JSON
- Inet/MacAddr
- `text` Search
- `postgis` Geometry

#### Currently supported pg features:
- inherits
- composite type (`basic`)
- aggregate functions
- window functions
- Large Object
- Pg_trgm


** _Tested on `PostgreSQL 11/12/13/14` with `Slick 3.6.0`._  
** _`Java 8`+ is required (except for `play-json` addon, which requires `java 11`+)._  



Usage
------
Before using it, you need integrate it with PostgresDriver maybe like this:

```scala
import com.github.tminglei.slickpg._


trait MyPostgresProfile extends ExPostgresProfile
                          with PgArraySupport
                          with PgDate2Support
                          with PgRangeSupport
                          with PgHStoreSupport
                          with PgPlayJsonSupport
                          with PgSearchSupport
                          with PgPostGISSupport
                          with PgNetSupport
                          with PgLTreeSupport {
  def pgjson = "jsonb" // jsonb support is in postgres 9.4.0 onward; for 9.3.x use "json"

  // Add back `capabilities.insertOrUpdate` to enable native `upsert` support; for postgres 9.5+
  override protected def computeCapabilities: Set[slick.basic.Capability] =
    super.computeCapabilities + slick.jdbc.JdbcCapabilities.insertOrUpdate

  override val api = MyAPI

  object MyAPI extends ExtPostgresAPI with ArrayImplicits
                                      with Date2DateTimeImplicitsDuration
                                      with JsonImplicits
                                      with NetImplicits
                                      with LTreeImplicits
                                      with RangeImplicits
                                      with HStoreImplicits
                                      with SearchImplicits
                                      with SearchAssistants {
    implicit val strListTypeMapper = new SimpleArrayJdbcType[String]("text").to(_.toList)
    implicit val playJsonArrayTypeMapper =
      new AdvancedArrayJdbcType[JsValue](pgjson,
        (s) => utils.SimpleArrayUtils.fromString[JsValue](Json.parse(_))(s).orNull,
        (v) => utils.SimpleArrayUtils.mkString[JsValue](_.toString())(v)
      ).to(_.toList)
  }
}

object MyPostgresProfile extends MyPostgresProfile

```

then in your codes you can use it like this:
```scala
import MyPostgresProfile.api._

class TestTable(tag: Tag) extends Table[Test](tag, Some("xxx"), "Test") {
  def id = column[Long]("id", O.AutoInc, O.PrimaryKey)
  def during = column[Range[Timestamp]]("during")
  def location = column[Point]("location")
  def text = column[String]("text", O.DBType("varchar(4000)"))
  def props = column[Map[String,String]]("props_hstore")
  def tags = column[List[String]]("tags_arr")

  def * = (id, during, location, text, props, tags) <> ((Test.apply _).tupled, Test.unapply)
}

object tests extends TableQuery(new TestTable(_)) {
  // will generate sql like:
  //   select * from test where id = ?
  def byId(ids: Long*) = tests
        .filter(_.id inSetBind ids)
        .map(t => t)
  // will generate sql like:
  //   select * from test where tags && ?
  def byTag(tags: String*) = tests
        .filter(_.tags @& tags.toList.bind)
        .map(t => t)
  // will generate sql like:
  //   select * from test where during && ?
  def byTsRange(tsRange: Range[Timestamp]) = tests
        .filter(_.during @& tsRange.bind)
        .map(t => t)
  // will generate sql like:
  //   select * from test where case(props -> ? as [T]) == ?
  def byProperty[T](key: String, value: T) = tests
        .filter(_.props.>>[T](key.bind) === value.bind)
        .map(t => t)
  // will generate sql like:
  //   select * from test where ST_DWithin(location, ?, ?)
  def byDistance(point: Point, distance: Int) = tests
        .filter(r => r.location.dWithin(point.bind, distance.bind))
        .map(t => t)
  // will generate sql like:
  //   select id, text, ts_rank(to_tsvector(text), to_tsquery(?))
  //   from test where to_tsvector(text) @@ to_tsquery(?)
  //   order by ts_rank(to_tsvector(text), to_tsquery(?))
  def search(queryStr: String) = tests
        .filter( t => {tsVector(t.text) @@ tsQuery(queryStr.bind)})
        .map(r => (r.id, r.text, tsRank(tsVector(r.text), tsQuery(queryStr.bind))))
        .sortBy(_._3)
}

...
```

_p.s. The code samples above are for `Slick` Lifted Embedding SQL. Aside from that, `slick-pg` also supports `Slick` Plain SQL--for details and usages please refer to source code and tests._


Configurable type/mappers
-------------------------
Since v0.2.0, `slick-pg` started to support configurable type/mappers.

Here's the related technical details:
> All pg type oper/functions related codes and some core type mapper logics were extracted to a new sub project "slick-pg_core", and the oper/functions and type/mappers binding related codes were retained in the main project "slick-pg".

**So, if you need bind different scala type/mappers to a pg type oper/functions, you can do it as "slick-pg" currently did.**


#### Built in supported type/mappers

| scala Type                                                                                               | pg Type               | dev 3rd-party library dependency     |
|----------------------------------------------------------------------------------------------------------| --------------------- |--------------------------------------|
| List[T]                                                                                                  | ARRAY                 | no 3rd party dependencies            |
| `java.sql` Date <br> Time<br> Timestamp<br> slickpg Interval<br> Calendar                                | date<br> time<br> timestamp<br> interval<br> timestamptz | no 3rd party dependencies            |
| `java.time` LocalDate<br> LocalTime<br> LocalDateTime<br> Duration<br> ZonedDateTime <br> OffsetDateTime | date<br> time<br> timestamp<br> interval<br> timestamptz <br> timestamptz | (built-in) no 3rd party dependencies |
| `joda` LocalDate<br> LocalTime<br> LocalDateTime<br> Period<br> DateTime                                 | date<br> time<br> timestamp<br> interval<br> timestamptz | `joda-time` v2.12.7                  |
| `scala` Enumeration                                                                                      | enum                  | no 3rd party dependencies            |
| `slickpg` Range[T]                                                                                       | range                 | no 3rd party dependencies            |
| `slickpg` LTree                                                                                          | ltree                 | no 3rd party dependencies            |
| Map[String,String]                                                                                       | hstore                | no 3rd party dependencies            |
| `slickpg` InetString                                                                                     | inet                  | no 3rd party dependencies            |
| `slickpg` MacAddrString                                                                                  | macaddr               | no 3rd party dependencies            |
| `slickpg` JsonString                                                                                     | json                  | no 3rd party dependencies            |
| `json4s` JValue                                                                                          | json                  | `json4s` v4.0.7                      |
| `play-json` JsValue                                                                                      | json                  | `play-json` v2.10 or v3.0            |
| `spray-json` JsValue                                                                                     | json                  | `spray-json` v1.3.5                  |
| `argonaut json` Json                                                                                     | json                  | `argonaut` v6.3.9                    |
| `circe json` Json                                                                                        | json                  | `circe` v0.14.6                      |
| `uPickle json` Json                                                                                      | json                  | `uPickle` v3.1.4                     |
| `jawn json` Json                                                                                         | json                  | `jawn` v1.5.1                        |
| (TsQuery+TsVector)                                                                                       | `text` search         | no 3rd party dependencies            |
| `jts` Geometry                                                                                           | `postgis` geometry    | `jts` v1.14.0                        |
| `locationtech's jts` Geometry                                                                            | `postgis` geometry    | locationtech's `jts` v1.19.0         |

_**Warning:** When your work with time data that contain Timezone, be wary of your postgres configuration. By default `ZonedDateTime` in Java 8 contains more information than `timestamptz` in Postgres. As a consequence, when you store a `ZonedDateTime` you are not guaranteed to get the same timezone as the original class instance. Prefer `OffsetDateTime` instead. cf [Issue #248](https://github.com/tminglei/slick-pg/issues/248)_

Details
------------------------------
- Array's [oper/functions](https://github.com/tminglei/slick-pg/tree/master/core/src/main/scala/com/github/tminglei/slickpg/array "Array's oper/functions"), usage  [cases](https://github.com/tminglei/slick-pg/blob/master/src/test/scala/com/github/tminglei/slickpg/PgArraySupportSuite.scala "test cases")
- JSON's [oper/functions](https://github.com/tminglei/slick-pg/tree/master/core/src/main/scala/com/github/tminglei/slickpg/json "JSON's oper/functions"), usage cases for [json4s](https://github.com/tminglei/slick-pg/blob/master/addons/json4s/src/test/scala/com/github/tminglei/slickpg/PgJson4sSupportSuite.scala "test cases"), [play-json](https://github.com/tminglei/slick-pg/blob/master/addons/play-json/src/test/scala/com/github/tminglei/slickpg/PgPlayJsonSupportSuite.scala "test cases"), [spray-json](https://github.com/tminglei/slick-pg/blob/master/addons/spray-json/src/test/scala/com/github/tminglei/slickpg/PgSprayJsonSupportSuite.scala "test cases"), [upickle json](https://github.com/tminglei/slick-pg/blob/master/addons/upickle-json/src/test/scala/com/github/tminglei/slickpg/PgUPickleJsonSupportSuite.scala "test cases") and [argonaut json](https://github.com/tminglei/slick-pg/blob/master/addons/argonaut/src/test/scala/com/github/tminglei/slickpg/PgArgonautSupportSuite.scala "test cases")
- Date/Time's [oper/functions](https://github.com/tminglei/slick-pg/tree/master/core/src/main/scala/com/github/tminglei/slickpg/date "Date/Time's oper/functions"), usage cases for [java date](https://github.com/tminglei/slick-pg/blob/master/src/test/scala/com/github/tminglei/slickpg/PgDateSupportSuite.scala "test cases"), [java 8 date](https://github.com/tminglei/slick-pg/blob/master/src/test/scala/com/github/tminglei/slickpg/PgDate2SupportSuite.scala "test cases") and [joda time](https://github.com/tminglei/slick-pg/blob/master/addons/joda-time/src/test/scala/com/github/tminglei/slickpg/PgDateSupportJodaSuite.scala "test cases")
- Enum's [oper/functions](https://github.com/tminglei/slick-pg/tree/master/core/src/main/scala/com/github/tminglei/slickpg/enums "Enum's oper/functions"), usage [cases](https://github.com/tminglei/slick-pg/blob/master/src/test/scala/com/github/tminglei/slickpg/PgEnumSupportSuite.scala "test cases")
- Range's [oper/functions](https://github.com/tminglei/slick-pg/tree/master/core/src/main/scala/com/github/tminglei/slickpg/range "Range's oper/functions"), usage [cases](https://github.com/tminglei/slick-pg/blob/master/src/test/scala/com/github/tminglei/slickpg/PgRangeSupportSuite.scala "test cases")
- HStore's [oper/functions](https://github.com/tminglei/slick-pg/tree/master/core/src/main/scala/com/github/tminglei/slickpg/hstore "HStore's oper/functions"), usage [cases](https://github.com/tminglei/slick-pg/blob/master/src/test/scala/com/github/tminglei/slickpg/PgHStoreSupportSuite.scala "test cases")
- LTree's [oper/functions](https://github.com/tminglei/slick-pg/tree/master/core/src/main/scala/com/github/tminglei/slickpg/ltree "LTree's oper/functions"), usage [cases](https://github.com/tminglei/slick-pg/blob/master/src/test/scala/com/github/tminglei/slickpg/PgLTreeSupportSuite.scala "test cases")
- PgTrgm's [oper/functions](https://github.com/tminglei/slick-pg/blob/master/core/src/main/scala/com/github/tminglei/slickpg/trgm "PgTrgm's oper/functions"), usage [cases](https://github.com/tminglei/slick-pg/blob/master/core/src/test/scala/com/github/tminglei/slickpg/trgm/PgTrgmSupportSuite.scala "test cases")
- Inet/MacAddr's [oper/functions](https://github.com/tminglei/slick-pg/tree/master/core/src/main/scala/com/github/tminglei/slickpg/net "net's oper/functions"), usage [cases](https://github.com/tminglei/slick-pg/blob/master/src/test/scala/com/github/tminglei/slickpg/PgNetSupportSuite.scala "test cases")
- Search's [oper/functions](https://github.com/tminglei/slick-pg/tree/master/core/src/main/scala/com/github/tminglei/slickpg/search "Search's oper/functions"), usage [cases](https://github.com/tminglei/slick-pg/blob/master/src/test/scala/com/github/tminglei/slickpg/PgSearchSupportSuite.scala "test cases")
- Geometry's [oper/functions](https://github.com/tminglei/slick-pg/tree/master/core/src/main/scala/com/github/tminglei/slickpg/geom "Geometry's oper/functions"), usage cases for [postgis](https://github.com/tminglei/slick-pg/blob/master/addons/jts/src/test/scala/com/github/tminglei/slickpg/PgPostGISSupportSuite.scala "test cases")
- `basic` Composite type [support](https://github.com/tminglei/slick-pg/tree/master/core/src/main/scala/com/github/tminglei/slickpg/composite "Composite type Support"), usage [cases](https://github.com/tminglei/slick-pg/blob/master/src/test/scala/com/github/tminglei/slickpg/PgCompositeSupportSuite.scala "test cases")
- Aggregate [functions](https://github.com/tminglei/slick-pg/tree/master/core/src/main/scala/com/github/tminglei/slickpg/agg "Aggregate functions"), usage [cases](https://github.com/tminglei/slick-pg/blob/master/core/src/test/scala/com/github/tminglei/slickpg/PgAggFuncSupportSuite.scala)
- Window [functions](https://github.com/tminglei/slick-pg/tree/master/core/src/main/scala/com/github/tminglei/slickpg/window), usage [cases](https://github.com/tminglei/slick-pg/blob/master/core/src/test/scala/com/github/tminglei/slickpg/PgWindowFuncSupportSuite.scala)



Install
-------
To use `slick-pg` in [sbt](http://www.scala-sbt.org/ "slick-sbt") project, add the following to your project file:
```scala
libraryDependencies += "com.github.tminglei" %% "slick-pg" % "0.23.0"
```

> If you need `joda-time` support, pls append dependency:
```scala
libraryDependencies += "com.github.tminglei" %% "slick-pg_joda-time" % "0.23.0"
```

> If you need `jts` geom support, pls append dependency:
```scala
libraryDependencies += "com.github.tminglei" %% "slick-pg_jts" % "0.23.0"
or
libraryDependencies += "com.github.tminglei" %% "slick-pg_jts_lt" % "0.23.0"
```

> If you need `json4s` support, pls append dependency:
```scala
libraryDependencies += "com.github.tminglei" %% "slick-pg_json4s" % "0.23.0"
```

> If you need `play-json` support, pls append dependency:
```scala
libraryDependencies += "com.github.tminglei" %% "slick-pg_play-json" % "0.23.0"
or
libraryDependencies += "com.github.tminglei" %% "slick-pg_play-json3" % "0.23.0"
```

> If you need `spray-json` support, pls append dependency:
```scala
libraryDependencies += "com.github.tminglei" %% "slick-pg_spray-json" % "0.23.0"
```

> If you need `argonaut json` support, pls append dependency:
```scala
libraryDependencies += "com.github.tminglei" %% "slick-pg_argonaut" % "0.23.0"
```

> If you need `circe json` support, pls append dependency:
```scala
libraryDependencies += "com.github.tminglei" %% "slick-pg_circe-json" % "0.23.0"
```

> If you need `uPickle json` support, pls append dependency:
```scala
libraryDependencies += "com.github.tminglei" %% "slick-pg_upickle-json" % "0.23.0"
```


Or, in [maven](http://maven.apache.org/ "maven") project, you can add `slick-pg` to your `pom.xml` like this:
```xml
<dependency>
    <groupId>com.github.tminglei</groupId>
    <artifactId>slick-pg_2.13</artifactId>
    <version>0.23.0</version>
</dependency>
<!-- other addons if necessary -->
...
```

**Notes:**
- the plugins' code were ever merged to the main project and published in an all-in-one jar from `slick-pg` v0.7.0, to easy usage, but I restored to publish them as independent jars from `slick-pg` v0.10.0, because of the issue pointed out by @timcharper in #183.
- plugin `slick-pg_date2` was merged into major jar from `slick-pg` v0.15.0, so you needn't add it as a depencensy any more.



Build instructions
------------------
`slick-pg` uses SBT for building and requires Java 8, since it provides support for `java.date` in addon `date2`. Assume you have already installed SBT, then you can simply clone the git repository and build `slick-pg` in the following way:
```
./sbt update
./sbt compile
```

_To run the test suite, you need:_
- create a user 'test' and db 'test' on your local postgres server, and
- the user 'test' should be an super user and be the owner of db 'test'

Then you can run the tests like this:
```
./sbt test
```
_ps: in the code of unit tests, the `slick` database is setup like this:_
```scala
val db = Database.forURL(url = "jdbc:postgresql://localhost/test?user=postgres", driver = "org.postgresql.Driver")
```


License
-------
Licensing conditions (BSD-style) can be found in LICENSE.txt.
