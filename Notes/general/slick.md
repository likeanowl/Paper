### Introduction
Slick is FRM (Functional Relational Mapping) library for Scala that makes it easy to work with relational DBs
Main points:
- Functional
- Async
- Object model and plain SQL
- Typesafe queries
- Interacts with DB only when required

Despite using objects, it is not ORM. Slick allows to execute SQL requests explicitly. Slick supports different relational DBs.


#### Addition to project
- Dependency on slick and db driver
- Application.conf
    ```conf
    dbProperties = {
        connectionPool = disabled
        url = "jdbc:db:url"
        driver = "org.h2.Driver"
        keepAliveConnection = true
    }
    ```
- Init database connection
    ```scala
    import slick.jdbc.H2Profile.api._

    private val db = Database.forConfig("dbProperties")
    ```

#### Examples
```scala
object FstExample {
    //Presents simple table structure
    // Additional attributes could be stated for fields, such as PrimaryKey
    class MessageTable(tag: Tag) extends Table[(Long, String)](tag, "message") {
        def id: Rep[Long] = column[Long]("id", O.PrimaryKey, O.AutoInc)
        def text: Rep[String] = column[String]("text")

        // Mapping of description columns to columns
        override def * : ProvenShape[(Long, String)] = (id, text)
    }

    def main(args: Array[String]): Unit = {
        // ---
        // Creating connection
        val db = Database.forConfig("dbProperties)
        // Query entites
        // Until wrapped in action, it is just a query
        val messages = TableQuery[MessageTable]

        val setupAction = DBIO.seq( //actions
            messages.schema.create, //create table
            messages += (OL, "test) // insert record
        )
        // ---
        // This whole section does not init any SQL queries
        try {
            val setupFuture = db.run(setupAction) // running actions 
            Await.ready(setupFuture, Duration.Inf)

            val selectFuture: Future[Seq[(Long, String)]] = db.run(messages.result)
            Await.result(selectFuture, Duration.Inf).foreach(println)
        } finally db.close()
    }
}
```

#### Basic concepts
- Profile - implements RDBMS-specific features
- Database - encapsulates the resources required for db connection
- Query - is used to build single SQL-query
- Database IO Action - is used to build sequences of queries
- Future - is used to transform the async result of running a DBIO Action

#### Table definition
- Table describes table schema
    ```scala
    abstract class Table[T] {
        def * : ProvenShape[T] // converts DB entities to mapped objects 
                               //and vice versa
    }
    ```
- TableQuery - represents the actual database table (select, update, join, ...)

#### Example 2
```scala
object FstExample {
    //Presents simple table structure
    // Additional attributes could be stated for fields, such as PrimaryKey
    case class Message(text: String, id: Long = 0L)
    class MessageTable(tag: Tag) extends Table[Message](tag, "message") {
        def id: Rep[Long] = column[Long]("id", O.PrimaryKey, O.AutoInc)
        def text: Rep[String] = column[String]("text")

        // Mapping of description columns to columns
        override def * : ProvenShape[(Long, String)] = 
            (text, id) <> (Message.tupled, Message.unapply)
    }

    private lazy val db = Database.forConfig("dbProperties)

    def execute[T] (action: DBIO[T]): T = Await.result(db.run(action), Duration.Inf)

    def main(args: Array[String]): Unit = {
        // Query entites
        val messages = TableQuery[MessageTable]
        execute(messages.schema.create)
        execute(messages += Mesage("Text"))
        execute(messages.result).foreach(println)
        db.close()
    }
}
```

#### More attributes
- Primary keys
- Foreign Keys
- Indexes

#### Selection of data
```scala
messages
    .filter(_.category === "Job")
    .sortBy(_.size)
    .map(m => (m.id, m.text))

// SELECT id, text
// FROM Messages
// WHERE category = 'Job'
// ORDER BY 'size"
```

it also could be performed via `for-comprehension`:
```scala
for {
    message <- messages
    if message.size < 900L
} yield message
```

#### Debugging queries
`Query` allows to execute certain methods for debugging:
- Query.selectStatement
- Query.deleteStatement
- Query.updateStatement
- Query.insertStatement

#### Joining
- Applicative join -- use explicit join method
    - join - inner join
    - joinLeft - left outer
    - joinRigh - right outer
    - joinFull - full outer

- Monadic Join -- flatMap as a way to join tables

Both types could be combined, let's see example:
```scala
object Example {
    case class User(id: Long, name: String)
    case class Message(text: String ,senderId: Long, id: Long = 0L)
    class UserTable(tag: Tag) extends Table[User](tag, "user") {
        def id: Rep[Long] = column[Long]("id, O.PrimaryKey)
        def name: Rep[String] = column[String]("name")

        override def * : ProvenShape[User] = (id, name).mapTo[User]
    }

    private lazy val users = TableQuery[UserTable]

    class MessageTable(tag: Tag) extends Table[Message](tag, "message") {
        def id: Rep[Long] = column[Long]("id", O.PrimaryKey, O.AutoInc)
        def senderId: Rep[Long] = column[Long]("sender_id")
        def text: Rep[String] = column[String]("text")
        // foreign key on senderId attribute
        def sender = foreignKey("sender_fk", senderId, users)(_.id)
        override def * : ProvenShape[Message] = (text, senderId, id).mapTo[Message]
    }

    private lazy val messages = TableQuery[MessageTable]
    private lazy val db = Database.forConfig("dbProperties")

    def execute[T](aciton: DBIO[T]): T = Await.result(db.run(action), Duration.Inf)

    def main(args: Array[String]): Unit = {
        val init = DBIO.seq(
            users.schema.creat,
            messages.schema.create,
            users ++= SEq(User(1, "User 1"), USer(2, "User 2")),

        )

        execute(init)

        val monadicJoin = for {
            msg <- messages
            // will be implicitly joined bc foreign key
            sender <- msg.sender if sender.id === 1L
        } yield (sender.name, msg.text)

        execute(monadicJoin.result).foreach(println)

        val anotherMonadicJoin = for {
            msg <- messages
            user <- users if user.id === msg.senderId && user.id === 1L
        } yield (user.name, msg.text)

        val applicativeJoin = messages
            .join(users).on(_.senderId === _.id)
            .filter { case (m, u) => u.id === 1L }
            .map { case (m, u) => (u.name, m.text) }

        execute(applicativeJoin.result).foreach(println)
    }
}
```

#### Modifying operations
##### Inserting rows
- Inserting single row
    ```scala
    val action = messages += Message("Text")
    ```
- Return PK value after insertion
    ```scala
    val action = messages forceInsert Message("Text")
    ```
- Inserting specific columns
    ```scala
    val action = messages.map(_.text) += "Text"
    ```
- Multiple rows
    ```scala
    messages ++= Seq(Message("From user 1"), ...)
    ```

##### Deleting rows
- Delete rows
    ```scala
    val action = messages.filter(_.senderId === 2L).delete
    ```
- Updating a singe field
    ```scala
    val query = messages.filter(_.senderId === 2L).map(_.text)
    val updateAction = query.update("New text")
    ```
- Update multiple fields
    ```scala
    val action = messages.filter(_.senderId === 2L)
        .map(m => (m.text, m.senderId)).update("New text", 3L)
    ```

##### Combining actions
These actions are not atomic by default (no transaction). In order to run operations in transaction we need to state this explicitly.
- `andThen` (>>)
    combined action are run sequentially (Last value returned)
    ```scala
    val action: DBIO[Int] = messages.delete andThen messages.size.result
    ```
- `DBIO.seq`
    even last value is discarded (`Unit`)
    ```scala
    val action: DBIO[Unit] = DBIO.seq(messages.delete, messages.size.result)
    ```
- `map/flatMap` (requires `ExecutionContext` because Slick could not guess our actions and therefore default Slick.ec is not suitable bc we could possibly lock execution, allows sequention of operations)
    ```scala
    import scala.concurrent.ExecutionContext.Implicits.global
    def insert(count: Int) = messages += Message(s"I removed $count messages")
    val actionDelete: DBIO[Int] = messages.delete.flatMap(count => insert(count))
    ```

#### Transactions
Sequential execution of operations could be run atomic. For this we need to define transaction section
```scala
val transactional = (
    ... >>
    ... >>
    ...
).transactionally
```
DBIO.failed causes rollback.
Let's see transactions with Slick in action:
```scala
// simple transaction
val transactional = (
    (messages += Message("Message 1")) >>
    (messages += Message("Message 2")) >>
    messages.filter(_.text like "%2").delete >>
    messages.result
).transactionally

execute(transactional).foreach(println)
// transaction with rollback 
val transactionWithRollback = (
    messages.delete >>
    DBIO.failed(new RuntimeException("Roll back transaction"))
).transactionally

execute(transactionalWithRollback.asTry)
```

### Further:
- Mapping details
- Transaction details
- Native SQL
- GroupBy