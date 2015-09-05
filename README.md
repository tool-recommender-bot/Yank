## [![Yank](https://raw.githubusercontent.com/timmolter/Yank/develop/etc/Yank_64_64.png)](http://xeiam.com/yank) Yank
Ultra-Light JDBC Persistance Layer for Java Apps

## In a Nutshell

Never deal with the monotony and pitfalls of handling JDBC ResultSets and Connections again. Yank deals with connection pooling and table row to Java object mapping for you so you don't have to worry about it.

## Long Description

Yank is a very easy-to-use yet flexible SQL-centric persistence layer for JDBC-compatible databases build on top of org.apache.DBUtils. Yank is a different approach to the over-ORMing of Java persistence.
Rather than try to abstract away the SQL underneath, Yank assumes you want low level control over the SQL queries you execute. Yank wraps DBUtils,
hiding the nitty-gritty Connection and ResultSet handling behind a straight-forward proxy class: `Yank`. "Query" methods execute SELECT statements and return POJOs or a List of POJOs. "Execute"
methods execute INSERT, UPDATE, and DELETE (and other) statements. Recently, annotation-based column-field mapping, batch executing, column list querying and scalar querying has been added. Since version 3.0.0, Yank uses the
[Hikari connection pool](https://github.com/brettwooldridge/HikariCP) as its integrated connection pool.

## Features

 * [x] Apache 2.0 license
 * [x] ~13KB Jar
 * [x] Uses Apache DBUtils for JDBC
 * [x] Uses HikariCP for connection pooling
 * [x] Supports prepared statements
 * [x] Java object create, read, update, and delete (or CRUD) operations
 * [x] Automatic snake case (my_column_name) to camel case (myColumnName) mapping
 * [x] Automatic column name to field mapping via annotations
 * [x] Retrieving assigned auto-increment primary key ID for inserts
 * [x] Java object and object List querying
 * [x] Scalar querying
 * [x] Column List querying
 * [x] Batch execute
 * [x] Works with any JDBC-compliant database
 * [x] Write your own SQL statements
 * [x] Optionally store connection pool properties in a Properties file
 * [x] Optionally store SQL statements in a Properties file
 * [x] Java 5 and up

## Basic Example

```java
public static void main(String[] args) {

  // Connection Pool Properties
  Properties dbProps = new Properties();
  dbProps.setProperty("jdbcUrl", "jdbc:mysql://localhost:3306/Yank");
  dbProps.setProperty("username", "root");
  dbProps.setProperty("password", "");

  // setup data source
  Yank.setupDataSource(dbProps);

  // query book
  String sql = "SELECT * FROM BOOKS WHERE TITLE = ?";
  Object[] params = new Object[] { "Cryptonomicon" };
  Book book = Yank.queryBean(sql, Book.class, params);
  System.out.println(book.toString());

  // release data source
  Yank.releaseDataSource();
}
```

## Hide Those Properties Away!

```java
// Connection Pool Properties
Properties dbProps = PropertiesUtils.getPropertiesFromClasspath("MYSQL_DB.properties");

// // setup data source
Yank.setupDataSource(dbProps);
```
Why? Hardcoding properties is fine for something quick and dirty, but loading them from a file is generally more convenient and flexible. For example, you may have separate properties for unit tests, development and production deployments. BTW, you can load them from a path too with: `PropertiesUtils.getPropertiesFromPath(String fileName)`. At the bare minimum, you need to provide `username`, `password`, and `jdbcUrl` configuration properties.

## Hide Those SQL Statements Away!

```java
// SQL Statements in Properties file
Properties sqlProps = PropertiesUtils.getPropertiesFromClasspath("MYSQL_SQL.properties");
Yank.addSQLStatements(sqlProps);
// ...
String sqlKey = "BOOKS_CREATE_TABLE";
Yank.executeSQLKey(sqlKey, null);
```
Why? Sometimes it's nice to have all your SQL statements in one place. As an example see: [MYSQL_SQL.properties](https://github.com/timmolter/Yank/blob/develop/src/test/resources/MYSQL_SQL.properties). Also this allows you to swap databases easily without changing any code. Keep one for database type `X` and one for database type `Y`. BTW, to access the actual statements in the  properties file, you use the `Yank.*SQLKey(...)` methods in `Yank`. You can also add multiple properties files and they will be merged! If the SQL statement cannot be found, a `SQLStatementNotFoundException` runtime exception is thrown.

## Stay Organized! You Will Thank Yourself Later.
```java
public class Book {

  private String title;
  private String author;
  private double price;
  
    // default constructor

  // getters and setters
}
```
```java

public class BooksDAO {

  public static int insertBook(Book book) {

    Object[] params = new Object[] { book.getTitle(), book.getAuthor(), book.getPrice() };
    String SQL = "INSERT INTO BOOKS  (TITLE, AUTHOR, PRICE) VALUES (?, ?, ?)";
    return Yank.execute(SQL, params);
  }

  // ...

  public static List<Book> selectAllBooks() {

    String SQL = "SELECT * FROM BOOKS";
    return Yank.queryBeanList(SQL, Book.class, null);
  }
}
```
Why? By creating a DAO class and putting all methods related to a single database table in it, you have a single point of access to that table. In this example the **BooksDAO** corresponds to a table called **Books**, which contains rows of **Book** objects (a.k.a beans). Note that your beans **must** have the default, no args constructor.

## Annotate Class Fields
```java
public static class Book {

  private int id;
  @Column("TITEL")
  private String title;
  @Column("AUTOR")
  private String author;
  @Column("PREIS")
  private double price;
  
  // default constructor

  // getters and setters
}
```
The default automatic mapping from database row to Java objects happens when the object's field names match the table column names (not case-sensitive). Automatic snake case (my_column_name) to camel case (myColumnName) mapping is supported too. If that still isn't good enough, you can annotate the Java object's fields with a `Column` annotation.

## Insert and Receive the Assigned ID
```java
Object[] params = new Object[] { book.getTitle(), book.getAuthorName(), book.getPrice() };
String SQL = "INSERT INTO BOOKS (TITLE, AUTHOR, PRICE) VALUES (?, ?, ?)";
Long id = Yank.insert(SQL, params);
```
With a special `Yank.insert(...)` method, Yank will return the assigned auto-increment primary key ID. Note that you can alternatively use the `Yank.execute(...)` method for inserts, which returns the number of affected rows.

## Retrieve a Column as a List
```java
String SQL = "SELECT TITLE FROM BOOKS";
String columnName = "title";
List<String> bookTitles = Yank.queryColumnList(SQL, columnName, String.class, null);
```
With the `Yank.queryColumnList(...)` method you can retrieve a List containing objects matching column data type.

## Query a Scalar Value
```java
String SQL = "SELECT COUNT(*) FROM BOOKS";
long numBooks = Yank.querySingleScalar(SQL, Long.class, null);
```
With the `Yank.querySingleScalar(...)` method you can retrieve a single scalar value that matches the return type of the given SQL statement.
## Life's a Batch
```java
List<Book> books = new ArrayList<Book>();
// add books to list

Object[][] params = new Object[books.size()][];

for (int i = 0; i < books.size(); i++) {
  Book book = books.get(i);
  params[i] = new Object[] { book.getTitle(), book.getAuthor(), book.getPrice() };
}

String SQL = "INSERT INTO BOOKS (TITLE, AUTHOR, PRICE) VALUES (?, ?, ?)";
int numInsertedRows = Yank.executeBatch(SQL, params);
```

## Summary

Whether or not your app is a tiny scipt, a large webapp, or anything in between the main pattern to follow is the same:

1. Configure a connection pool: `Yank.setupDataSource(dbProps);`
1. Use Yank's methods: `Yank.execute(...) `,`Yank.executeBatch(...) `, `Yank.insert(...) `, `Yank.queryColumn(...) `, `Yank.queryObjectArrays(...) `, `Yank.queryBeanList(...) `, `Yank.queryBean(...) `, `Yank.queryScalar(...) `
1. Release the connection pool: ` Yank.releaseDataSource();`

For an example of Yank in action in a `DropWizard` web application see [XDropWizard](https://github.com/timmolter/XDropWizard).

Now go ahead and [study some more examples](http://xeiam.com/yank-example-code), [download the thing](http://xeiam.com/yank-change-log) and [provide feedback](https://github.com/timmolter/Yank/issues).

## Caveats

Yank was designed to be ultra-light and ultra-convenient and is philosophically different than most other competing libraries. Some "sacrifices" were made to stick to this design.

 * No multi-statement transaction service (This may be just fine for small to medium projects or to back a REST web application's API: POST, GET, PUT, and DELETE. These correspond to create, read, update, and delete (or CRUD) operations, respectively.)
 * Checked SQLExceptions are logged (SQL Exceptions are internally caught and logged. This is a heavily debated topic and many differing opinions exist. Yank, being ultra-light, catches and logs SQLExceptions.)
 * A Hikari connection pool is used behind the scenes (Generic DataSource integration isn't supported. If you just want a connection pool that works and don't care about the specific implementation this point is irrelevant.)

For many cases, the above features are not necessary, but that's for you to determine. If you are developing a critical banking application, you will probably need those features. For other applications where 100% data integrity isn't critical (such as [bitcoinium.com](https://bitcoinium.com/) for example), Yank's simplicity may be attractive. In return for the sacrifices, you write less code and your code will cleaner. Additionally, since `Yank`'s methods are `public static`, you can access it from anywhere in your application and not have to worry about passing around a reference to it. If you need those missing features, check out these projects similar to Yank: [sql2o](http://www.sql2o.org/) and [JDBI](http://jdbi.org/).

## Getting Started

### Non-Maven

Download Jar: http://xeiam.com/yank-change-log

#### Dependencies

* commons-dbutils.dbutils-1.6.0
* org.slf4j.slf4j-api-1.7.12
* com.zaxxer.HikariCP-java6-2.3.9
* a JDBC-compliant Connector jar

### Maven

The Yank release artifacts are hosted on Maven Central.

Add the Yank library as a dependency to your pom.xml file:
```xml
<dependency>
    <groupId>com.xeiam</groupId>
    <artifactId>yank</artifactId>
    <version>3.0.2</version>
</dependency>
```
For snapshots, add the following to your pom.xml file:
```xml
<repository>
  <id>sonatype-oss-snapshot</id>
  <snapshots/>
  <url>https://oss.sonatype.org/content/repositories/snapshots</url>
</repository>

<dependency>
    <groupId>com.xeiam</groupId>
    <artifactId>yank</artifactId>
    <version>3.0.3-SNAPSHOT</version>
</dependency>
```
## Building

#### general

    mvn clean package  
    mvn javadoc:javadoc  

#### maven-license-plugin

    mvn license:check
    mvn license:format
    mvn license:remove

## DropWizard Integration

If you want to integrate Yank into a DropWizard application, head over to [XDropWizard](https://github.com/timmolter/XDropWizard) and grab [YankManager.java](https://github.com/timmolter/XDropWizard/blob/master/src/main/java/com/xeiam/xdropwizard/manager/YankManager.java) and add a simple configuration to your DropWizard [myapp.yml](https://github.com/timmolter/XDropWizard/blob/master/xdropwizard.yml) file.

## Bugs
Please report any bugs or submit feature requests to [Yank's Github issue tracker](https://github.com/timmolter/Yank/issues).  

## Continuous Integration
[![Build Status](https://travis-ci.org/timmolter/Yank.png?branch=develop)](https://travis-ci.org/timmolter/Yank.png)  
[Build History](https://travis-ci.org/timmolter/Yank/builds)

## Donations
15MvtM8e3bzepmZ5vTe8cHvrEZg6eDzw2w  
