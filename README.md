rxjava-jdbc
============

Efficient execution, concise code, and functional composition of database calls
using JDBC and [RxJava](https://github.com/Netflix/RxJava/wiki) [Observable](http://netflix.github.io/RxJava/javadoc/rx/Observable.html). 

Status: *pre-alpha*

Be aware APIs are in major flux at the moment.

Features
--------------------------
* Functionally compose database queries run sequentially or in parallel 
* Queries may be only partially run or indeed never run due to subscription cancellations thus improving efficiency
* Concise code
* Method chaining just leads the way (once you are on top of the RxJava api of course!)
* All the RxJava goodness!
* Automatically maps query result rows into typed tuples or your own classes
* CLOB and BLOB handling is simplified greatly

The only runtime dependencies are [rxjava-core](https://github.com/Netflix/RxJava/tree/master/rxjava-core) and apache commons-io.

Continuous integration with Jenkins for this project is [here](https://xuml-tools.ci.cloudbees.com/). <a href="https://xuml-tools.ci.cloudbees.com/"><img  src="http://web-static-cloudfront.s3.amazonaws.com/images/badges/BuiltOnDEV.png"/></a>

Maven site reports are [here](http://davidmoten.github.io/rxjava-jdbc/index.html).

Getting started
--------------------
After using [RxJava](https://github.com/Netflix/RxJava/wiki) on a work project and being very impressed with 
it (even without Java 8 lambdas!), I wondered what it could offer for JDBC usage. The answer is lots!

Here's a simple example:
```java
Database db = new Database(url);
List<String> names = db.
		.select("select name from person where name > ? order by name")
		.parameter("ALEX")
		.getAs(String.class)
		.toList().toBlockingObservable().single();
System.out.println(names);
db.close();
```
output:
```
[FRED, JOSEPH, MARMADUKE]
```
Without using rxjava-jdbc the code is ugly mainly because of the pain of closing jdbc resources:
```java
Connection con = null;
PreparedStatement ps = null;
ResultSet rs = null;
try {
    con = DriverManager.getConnection(url);
	ps = con.prepareStatement("select name from person where name > ? order by name");
	ps.setObject(1, "ALEX");
	rs = ps.executeQuery();
	List<String> list = new ArrayList<String>();
	while (rs.next()) {
		list.add(rs.getString(1));
	}
	System.out.println(list);
} catch (SQLException e) {
	throw new RuntimeException(e);
} finally {
	if (rs != null)
		try {
			rs.close();
		} catch (SQLException e) {
		}
	if (ps != null)
		try {
			ps.close();
		} catch (SQLException e) {
		}
	if (con!=null) {
		try {
			con.close();
		} catch (SQLException e) {
		}
	}
}
```
Functional composition of JDBC calls
-----------------------------------------
Here's an example, wonderfully brief compared to normal JDBC usage:
 
```java
import com.github.davidmoten.rx.jdbc.Database;
import rx.Observable;

// use composition to find the first person alphabetically with
// a score less than the person with the last name alphabetically
// whose name is not XAVIER. Two threads and connections will be used.

Database db = new Database(connectionProvider);
Observable<Integer> score = db
		.select("select score from person where name <> ? order by name")
		.parameter("XAVIER")
		.getAs(Integer.class)
		.last();
String name = db
		.query("select name from person where score < ? order by name")
		.parameters(score)
		.getAs(String.class)
		.first()
		.toBlockingObservable().single();
assertEquals("FRED", name);
```

About BlockingObservable
----------------------------
You'll see ```toBlockingObservable()``` used in the examples in this page and in 
the unit tests but in your application code you should try to avoid using it. The most benefit 
from the reactive style is obtained by *not leaving the monad*. That is, stay in Observable land and make 
the most of it. Chain everything together and leave toBlockingObservable to 
an endpoint or better still just subscribe with an Observer.

Dependencies
--------------
You can setup chains of dependencies that will determine the order of running of queries. 
By default queries are run in parallel (to the limits of the current QueryContext) unless
you specify dependencies. 

To indicate that a query cannot be run before one or more other Observables
have been completed use the `dependsOn()` method. Here's an example:
```java
Observable<Integer> insert = db
		.update("insert into person(name,score) values(?,?)")
		.parameters("JOHN", 45)
		.getCount()
		.map(Util.<Integer> delay(500));
int count = db
		.select("select name from person")
		.dependsOn(insert)
		.get()
		.count()
		.toBlockingObservable().single();
assertEquals(4, count);
```

Mixing explicit and Observable parameters
------------------------------------------
Example:
```java
String name= db
	.query("select name from person where name > ?  and score < ? order by name")
	.parameter("BARRY")
	.parameters(Observable.from(100))
	.getAs(String.class)
	.first()
	.toBlockingObservable().single();
assertEquals("FRED",name);
```

Passing multiple parameter sets to a query
--------------------------------------------
Given a sequence of parameters, each chunk of parameters will be run with the query and the results appended. 
In the example below there is only one parameter in the sql statement yet two parameters are specified.
This causes the statement to be run twice.

```java
List<Integer> list = 
	db.query("select score from person where name=?")
	    .parameter("FRED").parameter("JOSEPH")
		.getAs(Integer.class).toList().toBlockingObservable().single();
assertEquals(Arrays.asList(21,34),list);
```

Automap
------------------------------
Given this class:
```java
static class Person {
		private final String name;
		private final double score;
		private final Long dateOfBirth;
		private final Long registered;

		Person(String name, double score, Long dateOfBirth,
				Long registered) {
				...
```
We can get *rxjava-jdbc* to use reflection to auto map the fields in a result set to create an instance of Person:
```java
Observable<Person> persons = db
				.select("select name,score,dob,registered from person order by name")
				.autoMap(Person.class);
```
The main requirement is that the number of columns in the select statement must match 
the number of columns in a constructor of Person and that the column types can be 
automatically mapped to the types in the constructor.

Auto mappings
------------------
The automatic mappings below of objects are used in the ```autoMap()``` method and for typed ```getAs()``` calls.
* ```java.sql.Date```,```java.sql.Time```,```java.sql.Timestamp``` <==> ```java.util.Date```, ```long```
* ```java.sql.Blob``` <==> ```java.io.InputStream```, ```byte[]```
* ```java.sql.Clob``` <==> ```java.io.Reader```, ```String```

Tuples
---------------
Typed tuples can be returned in an Observable:
###Tuple2
```java
Tuple2<String, Integer> tuple = db
		.query("select name,score from person where name >? order by name")
		.parameter("ALEX").create()
		.execute(String.class, Integer.class).last()
		.toBlockingObservable().single();
assertEquals("MARMADUKE", tuple.value1());
assertEquals(25, (int) tuple.value2());
```
Similarly for Tuple3, Tuple4, Tuple5, Tuple6, Tuple7, and finally 
###TupleN
```java
TupleN<String> tuple = db
		.query("select name, lower(name) from person order by name")
		.create()
		.executeN(String.class).first()
		.toBlockingObservable().single();
assertEquals("FRED", tuple.values().get(0));
assertEquals("fred", tuple.values().get(1));
```

Large objects support
------------------------------
Blob and Clobs are straightforward to handle.

### Insert a Clob
Here's how to insert a String value into a Clob (*document* column below is of type CLOB):
```java
String document = ...
Observable<Integer> count = db
		.update("insert into person_clob(name,document) values(?,?)")
		.parameter("FRED")
		.parameter(document).getCount();
```
or using a java.io.Reader:
```java
Reader reader = ...;
Observable<Integer> count = db
		.update("insert into person_clob(name,document) values(?,?)")
		.parameter("FRED")
		.parameter(reader).getCount();
```
### Read a Clob
```java
Observable<String> document = db.select("select document from person_clob")
				.getAs(String.class);
```
or
```java
Observable<Reader> document = db.select("select document from person_clob")
				.getAs(Reader.class);
```
### Insert a Blob
Similarly for Blobs (*document* column below is of type BLOB):
```java
byte[] bytes = ...
Observable<Integer> count = db
		.update("insert into person_blob(name,document) values(?,?)")
		.parameter("FRED")
		.parameter(bytes).getCount();
```
### Read a Blob
```java
Observable<byte[]> document = db.select("select document from person_clob")
				.getAs(byte[].class);
```
or
```java
Observable<InputStream> document = db.select("select document from person_clob")
				.getAs(InputStream.class);
```
Transactions
------------------
Queries can be surrounded by beginTransaction and commit/rollback calls. The queries eventually run between the calls will 
use a single thread one by one and will all use the same Connection object. 

Queries within a transaction are constructed as normal using dependencies on or parameter lists from other queries
but the commit/rollback statement must also reference its dependencies.

A query can declare a dependency on the previous transaction completing by using the dependsOnLastTransaction() method.

Example:
```java
db.beginTransaction();
Observable<Integer> updateCount = db
		.update("update person set score=?")
		.parameter(99)
		.getCount();
// indicate dependency on  updateCount running before commit occurs
db.commit(updateCount);

//indicate dependency on last transaction before running this query
long count = db
		.select("select count(*) from person where score=?")
		.parameter(99)
		.dependsOnLastTransaction()
		.getAs(Long.class)
		.toBlockingObservable().single();
assertEquals(3, count);
```

Handlers
------------------------
You can specify a select handler and an update handler individually or one general 
handler. For example you could log all error events for all database queries.

The handlers operate globally for one ```Database``` object. An example is ```Handlers.LOG_ON_ERROR_HANDLER``` which intercepts an ```onError``` event and writes 
a SEVERE log line using ```java.util.logging``` and then passes the error on. To create handlers use the ```Database``` builder method:

```java
Database db = Database.builder(url).handler(Handlers.LOG_ON_ERROR_HANDLER).build();
//The call below will throw a RuntimeException but will also log the exception before throwing.
db.select("seeeelect name from person")
  .getAs(String.class).first().toBlockingObservable().single();
```

Closing the database
----------------------------
Once you've finished doing work with a ```Database``` run ```Database.close()``` or ```Database.closeAfter()```
to shutdown executors. If you need to perform some additional action to close your custom ```ConnectionProvider```
then you should do that also. 

 ```java
Database db = new Database(url);
List<String> names = db.
		.select("select name from person order by name")
		.getAs(String.class)
		.toList().toBlockingObservable().single();
System.out.println(names);

//close the database to shutdown executors 
db.close();
```

You can *stay in the monad* by making the closure of the database an Observable as well with dependencies.

```java
Observable<Integer> insert = db
		.update("insert into person(name,score,dob) values(?,?,?)")
		.parameters("JACKIE", 42, null).getCount();
db.closeAfter(insert).subscribe();
```
 