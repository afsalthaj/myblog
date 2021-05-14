---
title: Functional JDBC
---

## Everything with JDBC is difficult

Jdbc is just tough (personally), and all that we need is just querying something
from database returning something which can be used somewhere else. Not a great business
story, but the core of most of the business stories.

If you repeat the code of using raw jdbc functionalities multiple times (lots and lots of patterns of usages), 
probably you may abstract it in your code 
such that the abstraction will handle (basic) resource handling + querying.

So, this below blog/code is just that abstraction. Nothing more, and nothing less.

PS: If you are on a significantly large webservice with lots of database operations, Doobie is a great choice.

 Let's define a smart constructor for `Jdbc` `Connection`

### JdbcConnection

```scala

  import cats.effect.Resource
  import cats.effect.Sync
  import java.sql.Connection

  abstract sealed case class JdbcConnection[F[_]](resource: Resource[F, Connection])

  object JdbcConnection {
    final case class ConnectionInfo(
      username: String,
      password: String,
      url: String,
      driverName: DownstreamSystem.Jdbc.EngineType
    )

    def mk[F[_]: Sync](connectionInfo: ConnectionInfo): JdbcConnection[F] =
      new JdbcConnection(
        Resource
          .fromAutoCloseable(Sync[F].delay {
            Class.forName(connectionInfo.driverName.name)
            DriverManager.getConnection(connectionInfo.url, connectionInfo.username, connectionInfo.password)
          })
      ) {}
  }


```

The only way to construct `JdbcConnection` is `JdbcConnection.mk[IO]`, where `F` is substitued with `cats.effect.IO`
as an example. If you are not used to `F`, just consider that as "something" that represents a real `IO` operation.
We inject some power to `F` using typeclass constraint `F[_] : Sync`. Forget about it, if you are not so used to it.

Before we discuss how to use `JdbcConnection`, keep a note that, you are already in some `Resource` context.
In simple terms, anything that you do here after with `JdbcConnection` will be followed by closing the connection, guaranteed!

But before we start using the above abstraction, we need one more thing in place - `java.sql.Statement`, which
can be retrieved from `java.sql.Connection`, which is then used to query the database. i.e, `statement.execute(...)`

So, simple thinking leads us to follow just the same pattern as above for `java.sql.Statement`.

### JdbcStatement

Well, it seems `Statement` is all that we need for "most" of the operations. It seems this is our `Client`. Let's name
it as `SqlClient`

```scala

abstract sealed case class SqlClient[F[_]](resource: Resource[F, Statement])

object SqlClient {
  def mk[F[_]](connectionInfo: JdbcConnection.ConnectionInfo)(implicit F: Sync[F]): SqlClient[F] =
    JdbcConnection.mk(connectionInfo).statement
}

```

Well, I lied, there is no such function called `statement` in `JdbcConnection`. So let's add that.

```scala

  abstract sealed case class JdbcConnection[F[_]](resource: Resource[F, Connection]) {
    def statement(implicit F: Sync[F]): SqlClient[F] =
      new SqlClient(
        resource.flatMap(connection => Resource.fromAutoCloseable(Sync[F].delay(connection.createStatement())))
      ) {}
  }

```

Well, this code will not work unless we keep this code inside the companion object of `SqlClient`.
If this code needs to be in companion objecct, then the entire `JdbcConnection` should be with in the companion object
of `SqlClient`. And that, really, makes sense!.

```scala
 abstract sealed case class SqlClient[F[_]](resource: Resource[F, Statement])

 object SqlClient {
   def mk[F[_]](connectionInfo: JdbcConnection.ConnectionInfo)(implicit F: Sync[F]): SqlClient[F] =
     JdbcConnection.mk(connectionInfo).statement

   abstract sealed case class JdbcConnection[F[_]](resource: Resource[F, Connection]) {
     new SqlClient(
       resource.flatMap(connection => Resource.fromAutoCloseable(Sync[F].delay(connection.createStatement())))
     ) {}
   }

   object JdbcConnection {
     final case class ConnectionInfo(
       username: String,
       password: String,
       url: String,
       driverName: DownstreamSystem.Jdbc.EngineType
     )

     def mk[F[_]: Sync](connectionInfo: ConnectionInfo): JdbcConnection[F] =
       new JdbcConnection(
         Resource
           .fromAutoCloseable(Sync[F].delay {
            Class.forName(connectionInfo.driverName.name)
            DriverManager.getConnection(connectionInfo.url, connectionInfo.username, connectionInfo.password)
          })
      ) {}   
   }  
 }


```

I think that's our base. We still haven't looked at using any of this, but you can imagine it as
`SqlClient.mk[IO]`, but we can't run any query with it. 

So here is the (almost) final bit. Define a select query function in `SqlClient`

```scala

abstract sealed case class SqlClient[F[_]](resource: Resource[F, Statement]) {
  def select(
    query: String
  )(implicit F: Sync[F], C: Stream.Compiler[F, F]): F[List[Row]] =
    resource
      .use(
        r =>
          Sync[F]
            .delay(r.executeQuery(query))
            .flatMap(
              resultSet => ??? // I hate resultSet, let's see what we can do with ResultSet
            )
      )
 }

 object SqlClient {
  type Row = Map[String, AnyRef]

  // ...
 }

```

Well we haven't got everything in place. We just tried to implement `select`, and we know the return type is `F[List[Row]]`.
`Row` is as simple as `Map[String, AnyRef]`. Example: `Map("age" -> 20, "country`" : "usa")`.

Sure, how do we convert `java.sql.ResultSet` to `List[Row]` ?
Well, the best answer is, it should be from `java.sql.ResultSet` to `fs2.Stream[F, Row]`. Why?
There can be numerous number of rows and it cannot be collected to memory. Hence it's  a stream.
It's a stream under an effect `F` representing `IO`.

In functional programming terms, it is a co-recursion, which is recognising the fact that it is really an `unfold` operation,
where we build up the structure from an intitial ResultSet state (i.e, the famous resultSet.next()) 
until the state transition stops. Note: In certain cases, the transition never stops leading to infinite data!

This is probably the time, developers who are not familiar with Functional Programming has that moment of table-flip. 
However, I still request you to give me a bit more time before you flip.

Oversimplifying a few things here.

There are two main types of recursion. Unfolding, and folding.
Let's call it as "co-recursion" and "recursion" respectively.

A corecursion is still a recursion, but it is building the data. Example: Stream.
On the other hand, a non-co-recursion (recursion) will terminate the data to a single point. Example: Sum of a list of Int.

That's a bit wordy. Anyway, let's have our own `JdbcResultSet` similar to our above patterns.
Who wouldn't want a "consistency" in the code base!

```scala
  abstract sealed class JdbcResultSet(resultSet: ResultSet)

  object ResultSet {
    // Now we move the Row from SqlClient Companion object to here.
    // Placement of code at the right place is super important in my opinion.
    type Row = Map[String, AnyRef]

    def from(resultSet: ResultSet): JdbcResultSet =
      new JdbcResultSet(resultSet) {} 
  }

```

Let's build the `unfold` of `ResultSet` to form a `Stream[F, Row]` (collection of rows).
The best place to keep this function is the companion object of `JdbcResultSet`.


```scala
  object ResultSet {
    // Now we move the Row from SqlClient Companion object to here.
    // Placement of code is super important in my opinion.
    type Row = Map[String, AnyRef]

    def from(resultSet: ResultSet): JdbcResultSet =
      new JdbcResultSet(resultSet) {} 

    // Have a look at the difference between Stream.unfoldLoop and Stream.unfold
    // The latter skips the current chunk of data in the output stream if next state (resultSet.next) is absent.
    // The former keeps the current chunk of data in the output stream even if next state (resultSet.next)is absent.
    // Initial state is resultSet.next()
    // if present, return (row, resultSet.next().toOption)
    // if absent, return (row, None)
    // and the co-recursion continues.
    def rows[F[_]]: Stream[F, JdbcResultSet.Row] =
      Stream.unfoldLoop(resultSet.next())(
        hasMore =>
          if (hasMore) {
            val metadata: ResultSetMetaData =
              resultSet.getMetaData

            val columnNames =
              for (i <- 1 to metadata.getColumnCount) yield metadata.getColumnName(i)

            val row = 
              (for (c <- columnNames) yield c -> resultSet.getObject(c)).toMap

            (row, if (resultSet.next()) Some(true) else None)
          } else (Map.empty, None)
      )  
  }

```

I think we are done. Let's implement select again.

```scala

  def select(
    query: String
  )(implicit F: Sync[F], C: Stream.Compiler[F, F]): F[List[SqlClient.JdbcResultSet.Row]] =
    resource
      .use(
        r =>
          Sync[F]
            .delay(r.executeQuery(query))
            .flatMap(
              resultSet => SqlClient.JdbcResultSet.from(resultSet).rows.compile.toList
            )
      )

```

### We all love non-emptiness in type

Let's implement selectNonEmpty, that should return `NonEmptyList` of rows,
and the types tell you this infact, and you don't need to worry of empty result
at call-site.

```scala
  def selectNonEmpty(
    query: String
  )(implicit F: Sync[F], C: Stream.Compiler[F, F]): F[NonEmptyList[SqlClient.JdbcResultSet.Row]] =
    select(query)
      .flatMap(
        list =>
          NonEmptyList.fromList(list) match {
            case Some(value) =>
              Sync[F].pure(value)
            case None =>
              Sync[F].raiseError(new RuntimeException(s"Zero results returned for ${query}"))
          }
      )

```

### Let's type the stuff

Well, instead of a `Row`, why can't it be a proper type, example, a `Customer` case class.

If you are not familiar with `zio-config`, I kindly recommend refering the documentation.
https://zio.github.io/zio-config/

`Descriptor` below belongs to `zio-config`

```scala
  
  /**
   *
   * {{{
   *  case class Customer(age: Int)
   *  SqlClient.mk(connectionInfo).selectA[Customer](query)
   * }}}
   *
   * returns a non-empty list of Customer.
   */
  def selectA[A: Descriptor](query: String)(implicit F: Sync[F]): F[NonEmptyList[A]] =
    selectNonEmpty(query).flatMap(
      results =>
        results.traverse(
          row =>
            SqlClient.JdbcResultSet.Row.to[A](row) match {
              case Left(value)  => Sync[F].raiseError(new RuntimeException(value))
              case Right(value) => Sync[F].pure(value)
            }
        )
    )

```

```scala
  type Row = Map[String, AnyRef]

  object Row {
    def to[A: Descriptor](row: Row): Either[String, A] = {
      val source = ConfigSource.fromMap(row.mapValues(_.toString))
      read(descriptor[A] from source).leftMap(_.nonPrettyPrintedString)
    }
  }

```

### Let's play with resources

How do we handle the case where we have multiple small queries that should use the same `Statement`
(and hence the same `Connection` under the hood) ?

```scala

  def execute[A](queries: String*)(implicit F: Sync[F]): F[Unit] =
    resource
      .use(
        statement =>
          queries.toList.traverse[F, Unit](
            query => Sync[F].delay(statement.execute(query))
          )
      )
      .void

```

How do we have handle multiple long running queries and each one of them needs its
connection and statement ?

```scala
  // statement and then the connection will be closed after running each query
  def executeLongRunningQueries[A](queries: String*)(implicit F: Sync[F]): F[Unit] =
    queries.toList
      .traverse[F, Unit](query => resource.use(s => Sync[F].delay(s.execute(query))))
      .void

```

## Let's sum it up

```scala

import cats.effect.Resource
import cats.effect.Sync
import java.sql.DriverManager
import java.sql.Connection
import java.sql.ResultSet
import java.sql.Statement
import cats.syntax.traverse._
import cats.instances.list._
import fs2.Stream
import java.sql.ResultSetMetaData
import zio.config._
import zio.config.magnolia._
import cats.syntax.flatMap._
import cats.syntax.functor._
import cats.data.NonEmptyList
import com.thaj.safe.string.interpolator._
import cats.syntax.either._

abstract sealed case class SqlClient[F[_]](resource: Resource[F, Statement]) {

  /**
   * {{{
   *   SqlClient.mk(connectionInfo).select("select * from tablename")
   * }}}
   *
   * returns a list (can be empty list) of rows.
   */
  def select(
    query: String
  )(implicit F: Sync[F], C: Stream.Compiler[F, F]): F[List[SqlClient.JdbcResultSet.Row]] =
    resource
      .use(
        r =>
          Sync[F]
            .delay(r.executeQuery(query))
            .flatMap(
              resultSet => SqlClient.JdbcResultSet.from(resultSet).rows.compile.toList
            )
      )

  /**
   * {{{
   *   SqlClient.mk(connectionInfo).selectNonEmpty("select * from tablename")
   * }}}
   *
   * returns a non-empty list of rows.
   */
  def selectNonEmpty(
    query: String
  )(implicit F: Sync[F], C: Stream.Compiler[F, F]): F[NonEmptyList[SqlClient.JdbcResultSet.Row]] =
    select(query)
      .flatMap(
        list =>
          NonEmptyList.fromList(list) match {
            case Some(value) =>
              Sync[F].pure(value)
            case None =>
              Sync[F].raiseError(new RuntimeException(s"Zero results returned for ${query}"))
          }
      )

  /**
   *
   * {{{
   *  case class Customer(age: Int)
   *  SqlClient.mk(connectionInfo).selectA[Customer](query)
   * }}}
   *
   * returns a non-empty list of Customer.
   */
  def selectA[A: Descriptor](query: String)(implicit F: Sync[F]): F[NonEmptyList[A]] =
    selectNonEmpty(query).flatMap(
      results =>
        results.traverse(
          row =>
            SqlClient.JdbcResultSet.Row.to[A](row) match {
              case Left(value)  => Sync[F].raiseError(new RuntimeException(value))
              case Right(value) => Sync[F].pure(value)
            }
        )
    )

  /**
   *
   * {{{
   *   SqlClient.mk(connectionInfo).selectSingleton("select * from customers where customer_id = '1'")
   * }}}
   *
   * returns exactly one row.
   */
  def selectSingleton(query: String)(implicit F: Sync[F]): F[SqlClient.JdbcResultSet.Row] =
    selectNonEmpty(query).flatMap({
      case NonEmptyList(head, Nil) =>
        Sync[F].pure(head)
      case list =>
        Sync[F].raiseError(
          new RuntimeException(
            s"Multiple rows (size: ${list.size}) returned for the query ${query}. Only a single row is expected for this query."
          )
        )
    })

  /**
   * {{{
   *   case class Customer(..)
   *   SqlClient.mk(connectionInfo).selectSingletonA[Customer]("select * from customers where customer_id = '1'")
   * }}}
   *
   * returns exactly one `Customer`.
   */
  def selectSingletonA[A: Descriptor](query: String)(implicit F: Sync[F]): F[A] =
    selectSingleton(query).flatMap(
      row =>
        SqlClient.JdbcResultSet.Row.to[A](row) match {
          case Left(value)  => Sync[F].raiseError(new RuntimeException(value))
          case Right(value) => Sync[F].pure(value)
        }
    )

  /**
   * A single connection->statement for potentially multiple queries.
   * Note: These are not in a transaction.
   */
  def execute[A](queries: String*)(implicit F: Sync[F]): F[Unit] =
    resource
      .use(
        statement =>
          queries.toList.traverse[F, Unit](
            query => Sync[F].delay(statement.execute(query))
          )
      )
      .void

  /**
   * Each query will have its own connection->statement resource.
   */
  def executeLongRunningQueries[A](queries: String*)(implicit F: Sync[F]): F[Unit] =
    queries.toList
      .traverse[F, Unit](query => resource.use(s => Sync[F].delay(s.execute(query))))
      .void

  /**
   *  Delete table or view
   */
  def delete(objectName: String)(implicit F: Sync[F]): F[Unit] =
    execute(ss"DROP TABLE IF EXISTS ${objectName}".string)

}

object SqlClient {
  def mk[F[_]](connectionInfo: JdbcConnection.ConnectionInfo)(implicit F: Sync[F]): SqlClient[F] =
    JdbcConnection.mk(connectionInfo).statement

  // An immutable resultSet
  abstract sealed class JdbcResultSet(resultSet: ResultSet) {

    def rows[F[_]]: Stream[F, JdbcResultSet.Row] =
      //val newResultSet = resultSet
      Stream.unfoldLoop(resultSet.next())(
        hasMore =>
          if (hasMore) {
            val metadata: ResultSetMetaData =
              resultSet.getMetaData

            val columnNames =
              for (i <- 1 to metadata.getColumnCount) yield metadata.getColumnName(i)
            val row = (for (c <- columnNames) yield c -> resultSet.getObject(c)).toMap
            (row, if (resultSet.next()) Some(true) else None)
          } else (Map.empty, None)
      )
  }

  object JdbcResultSet {
    type Row = Map[String, AnyRef]

    def from(resultSet: ResultSet): JdbcResultSet =
      new JdbcResultSet(resultSet) {}

    object Row {
      def to[A: Descriptor](row: Row): Either[String, A] = {
        val source = ConfigSource.fromMap(row.mapValues(_.toString))
        read(descriptor[A] from source).leftMap(_.nonPrettyPrintedString)
      }
    }
  }

  abstract sealed case class JdbcConnection[F[_]](resource: Resource[F, Connection]) {
    def statement(implicit F: Sync[F]): SqlClient[F] =
      new SqlClient(
        resource.flatMap(connection => Resource.fromAutoCloseable(Sync[F].delay(connection.createStatement())))
      ) {}
  }

  object JdbcConnection {
    final case class ConnectionInfo(
      username: String,
      password: String,
      url: String,
      driverName: DownstreamSystem.Jdbc.EngineType
    )

    def mk[F[_]: Sync](connectionInfo: ConnectionInfo): JdbcConnection[F] =
      new JdbcConnection(
        Resource
          .fromAutoCloseable(Sync[F].delay {
            Class.forName(connectionInfo.driverName.name)
            DriverManager.getConnection(connectionInfo.url, connectionInfo.username, connectionInfo.password)
          })
      ) {}
  }
}


```

That's your functional JDBC API !
