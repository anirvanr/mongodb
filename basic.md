SQL vs NoSQL
```
1.	SQL databases use structured query language (SQL) for defining and manipulating data. SQL requires that you use predefined schemas to determine the structure of your data before you work with it. A NoSQL database, on the other hand, has dynamic schema for unstructured data.

2.	In most situations, SQL databases are vertically scalable, which means that you can increase the load on a single server by increasing things like CPU, RAM or SSD. NoSQL databases, on the other hand, are horizontally scalable. This means that you handle more traffic by sharding, or adding more servers in your NoSQL database.

3.	SQL databases are table-based, while NoSQL databases are either document-based, key-value pairs, graph databases or wide-column stores.
```

There are 4 basic types of NoSQL database. They are as follows: –
```
Key value store NoSQL database - Redis
Document store NoSQL database - CouchDB
Column store NoSQL database - Cassandra
Graph-based NoSQL database - Neo4J
```
MongoDB is a document-oriented (do) database, not a relational one. Non-relational means it does not store data in tables but in the form of JSON document.
A document is a set of key-value pairs.
Collection is a group of MongoDB documents.

features of mongodb:-
```
1. Document Oriented and NoSQL database
2. Sharding (Helps in Horizontal Scalability)
3. Schema less
4. The Ad hoc queries in MongoDB – it supports field, range queries, and regular expressions
5. It supports Master slave replication
6. Any field in a MongoDB document can be indexed
```

connect to local mongodb, default port: 27017 `mongo`
connect to remote mongodb `mongo host:port/db -u usr -p pwd`
mongodb connect URI format `mongodb://usr:pwd@host:port/db`
check the version `db.version()`
get stats about MongoDB server, type the command `db.stats()`
get data size and index size in MB `db.stats(1024*1024)`
show all databases `show dbs`
check your currently selected database, use the command `db`
create a database, say, testdb `use testdb`
add a collection `db.createCollection("cities")`
show all collections in a database `show collections`
insert a record in the collection `db.cities.insert({"name":"Paris","country":"France"})`
display list of records of a collection `db.cities.find().pretty()`
see one document from a collection `db.cities.findOne()`
drop the collection `db.cities.drop()`
remove all documents from a collection `db.cities.remove({})`
delete a database `use testdb` `db.dropDatabase()`

__Remove Vs Drop__

Remove: remove documents from a collection but doesn't get rid of the collection (or associated indexes)
Drop: remove documents/collection/indexes


__Save (upsert: update+insert) Vs Insert__

For save, If the document contains _id, it will take the document and replace the complete document having same _id, If not, it will insert.

Having _id:
```
set:PRIMARY> db.mycol.find()
{ "_id" : ObjectId("5c9b3241a6ee80bbaa7e1915"), "name" : "Bob" }
{ "_id" : ObjectId("5c9b59f47aa92200e26c79af"), "name" : "John" }
set:PRIMARY> db.mycol.save({"_id" : ObjectId("5c9b59f47aa92200e26c79af"), "name": "Johnny"})
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
set:PRIMARY> db.mycol.find()
{ "_id" : ObjectId("5c9b3241a6ee80bbaa7e1915"), "name" : "Bob" }
{ "_id" : ObjectId("5c9b59f47aa92200e26c79af"), "name" : "Johnny" }
```

Not Having _id:
```
set:PRIMARY> db.mycol.insert({"name": "Johnny"})
WriteResult({ "nInserted" : 1 })
set:PRIMARY> db.mycol.find()
{ "_id" : ObjectId("5c9b3241a6ee80bbaa7e1915"), "name" : "Bob" }
{ "_id" : ObjectId("5c9b59f47aa92200e26c79af"), "name" : "Johnny" }
{ "_id" : ObjectId("5c9b5a957aa92200e26c79b0"), "name" : "Johnny" }
```

Every document stored in MongoDB must have an "_id" key. The "_id" key’s value can be any type, but it defaults to an ObjectId. _id is `12 bytes hexadecimal number` unique for every document in a collection. 12 bytes are divided as follows:−
```
4 bytes timestamp
3 bytes machine id
2 bytes process id 
3 bytes incrementer
```
We can use __`namespace`__ to logically group and nest collections. For example : There can be one collection named `db.studytonight.users` to save user informations, then there can be others like `db.studytonight.forum.questions` and `db.studytonight.forum.answers` to store forum questions and answers respectively.

The database __`profiler`__ collects fine grained data about MongoDB queries longer than the specific threshold - threshold in milliseconds at which the database profiler considers a query slow. The database profiler writes all the data it collects to the system.profile collection so you can analyze them later.
Profiler has 3 profiling levels
```
0 - the profiler is off, does not collect any data. This is the default profiler level.
1 - collects profiling data for slow operations (slower than 100 milliseconds) only.
2 - collects profiling data for all database operations.
```
For example, the following command sets the profiling level for the current database to 1 and the slow operation threshold is 1000 milliseconds `db.setProfilingLevel(1, 1000)`. Database will log operations slower than 1000 milliseconds into `system.profile` collection. Now you can query for the data against this collection and analize: `db.system.profile.find().pretty()`.

get current profiling level `db.getProfilingLevel()`
check current profiling status `db.getProfilingStatus()`
set profiling for all operations `db.setProfilingLevel(2)`
```
> use test
> db.setProfilingLevel(2)
> db.sample.insert({_id:1 ,"text":"Sample Text"})
> db.system.profile.find({"op":"insert"}).sort({ts:-1}).limit(1).pretty()
{
	"op" : "insert",
	"ns" : "test.sample",
	"command" : {
		"insert" : "sample",
		"ordered" : true,
		"lsid" : {
			"id" : UUID("b7f74d71-1fce-40f4-8222-4bb6579f4672")
		},
		"$db" : "test"
	},
	"ninserted" : 1,
	"keysInserted" : 1,
	"numYield" : 0,
	"locks" : {
		"Global" : {
			"acquireCount" : {
				"r" : NumberLong(3),
				"w" : NumberLong(3)
			}
		},
		"Database" : {
			"acquireCount" : {
				"w" : NumberLong(2),
				"W" : NumberLong(1)
			}
		},
		"Collection" : {
			"acquireCount" : {
				"w" : NumberLong(2)
			}
		}
	},
	"responseLength" : 45,
	"protocol" : "op_msg",
	"millis" : 76,
	"ts" : ISODate("2019-04-01T06:02:04.428Z"),
	"client" : "127.0.0.1",
	"appName" : "MongoDB Shell",
	"allUsers" : [ ],
	"user" : ""
}
```
•	Op field stores the type of operation.
•	Ns field stores target database and collection name
•	Nreturned stores the number of documents returned by the query
•	Millis contains the actual time in milliseconds taken by this query to execute
•	Ts stores the timestamp of the query

NOTE: If you plan to use profiler in a production environment, then you should do proper testing because it can impact on your database throughput especially when you are logging all the queries i.e profiling level is set to 2. You can set you profiler on per database, for that you have to first select you database. And then execute profiler commands.

When the db.collection.find () function is used to search for documents in the collection, the result returns a pointer to the collection of documents returned which is called a __`cursor`__. By default, the cursor will be iterated automatically when the result of the query is returned.

show number of documents in the collection `db.books.find().count()`
limit the number of documents to return `db.books.find().limit(2)`
return the result set after skipping the first n number of documents `db.books.find().skip(2)`
sort the documents in a result set in ascending order of field values `db.books.find().sort({title : 1})`

__`Capped Collection`__: A collection created with a cap (limit on size and number of documents).
`db.createCollection("AuditTrail", {capped : true, size : 1000, max : 10 })`
Size determines the document size in bytes
Max determines maximum documents allowed.
Once the maximum size limit has reached and if we try to add more documents, the old documents get over ridden by new one.
Let's find out if the collection is capped or not: `db.AuditTrail.isCapped()`

Advantages of Capped Collections:
```
1. Queries do not need an index to return documents in insertion order due to which it provide higher insertion throughput.
2. Capped collections only allow updates that fit the original document size, which ensures a document does not change its location on disk.
```
Disadvantages of Capped Collections:
```
1. If the update operation increases the original size of the capped collection, the update operation fails.
2. We cannot delete documents from a capped collection. 
3. We cannot shard a capped collection.
```
The capped collections in MongoDB are suitable to store cache data, log information, or any other high volume data that need to be refreshed periodically.
Capped collections guarantee preservation of the insertion order.
For example, the "oplog.rs" collection that stores a log of the operations in a replica set uses a capped collection. If there is an existing collection which you want to change it to capped, you can do it by using the following command:`db.runCommand({"convertToCapped":"posts",size:10000})`

MongoDB __`projection`__ is a query where we can specify the fields we would like to have returned. We can do projection in MongoDB by adding a 0 or 1 next to fields name after including in a query.

Without projection:
```
> db.examples.find({subject:"Joe"})
{ "_id" : ObjectId("5c99bb5726cc3a7628d1cd14"), "subject" : "Joe", "content" : "best friend", "likes" : 60, "year" : 2015, "language" : "english" }
```
With Projection
```
> db.examples.find({subject:"Joe"},{likes:1})
{ "_id" : ObjectId("5c99bb5726cc3a7628d1cd14"), "likes" : 60 }
```
ObjectId field is automatically included and can be excluded by setting a 0 for this field.
```
> db.examples.find({subject:"Joe"},{_id:0,likes:1})
{ "likes" : 60 }
```



