# Mongodb Shard
Sharding is a concept in MongoDB, which splits large data sets into small data sets across multiple MongoDB instances. 
A sharded setup of mongodb requires the following:
```
•	Mongodb Configuration server – this stores the cluster’s metadata
•	mongos instance (shard controller )– this is the router and routes the queries to different shards based on the sharding key.
•	Individual mongodb instances – these act as the shards. 
```
##Key points:
```
•	Sharding is performed at the collection level.
•	A database is a mixture of sharded and unsharded collections in MongoDB.
•	MongoDB sharded collections are partitioned and distributed in clusters.
•	MongoDB unsharded collections are stored on a primary shard.
•	A collection cannot be un-sharded once it has been sharded
•	The primary shard is different for each database in MongoDB.
•	The primary shard has no relation to the primary in a replica set.
•	To distribute the documents in a collection, MongoDB partitions the collection using the shard key.
•	A sharded collection can have only one shard key.
•	The choice of shard key affects the performance, efficiency, and scalability of a sharded cluster.
•	MongoDB partitions sharded data into chunks.
•	The default chunk size in MongoDB is 64 megabytes.
•	Sometimes chunks cannot be broken up and continue to grow beyond the configured size (i.e., 64 MB). The balancer cannot move it. These chunks remain on the particular shard and are called jumbo chunks.
•	There are two types of strategies offers by MongoDB Sharding: Hashed Sharding and Ranged Sharding.
```
Installing mongodb on 5 machines with the following deamon configurations:

| Host | Mongo Role |
| --- | --- |
| router.mongo.cw.com | Router(mongos), Application Server, Arbiter (Shard1), Arbiter (Shard2), Config1, Config2, Config3 |
| shard1r1.mongo.cw.com | shard1 replica primary |
| shard1r2.mongo.cw.com | shard1 replica secondary |
| shard2r1.mongo.cw.com | shard2 replica primary |
| shard2r2.mongo.cw.com | shard2 replica secondary |

##Install Mongo
Download and install mongodb on all machines:


##Configure ReplicaSet1 and tell it to be shard 1
####Configure Replica Primary on shard1r1.mongo.cw.com

```
mkdir -p /data/shard1 /var/log/mongodb
/opt/mongodb/bin/mongod --replSet shard1 --logpath "/var/log/mongodb/shard1.log" --dbpath /data/shard1 --port 27017 --fork --shardsvr --bind_ip shard1r1.mongo.cw.com
```

####Configure Replica Secondary on shard1r2.mongo.cw.com
 
```
mkdir -p /data/shard1 /var/log/mongodb
/opt/mongodb/bin/mongod --replSet shard1 --logpath "/var/log/mongodb/shard1.log" --dbpath /data/shard1 --port 27017 --fork --shardsvr --bind_ip shard1r2.mongo.cw.com
```

###Configure Replica Arbiter on router.mongo.cw.com

```
mkdir -p /data/shard1/arbiter /var/log/mongodb
/opt/mongodb/bin/mongod --replSet shard1 --logpath "/var/log/mongodb/arbiter_shard1.log" --dbpath /data/shard1/arbiter --port 47018 --fork --bind_ip router.mongo.cw.com
```

####Initialize replica set 1 from mongo shell from replica primary 'shard1r1.mongo.cw.com'

Before, doing so wait until all the replicas came up online

```
/opt/mongodb/bin/mongo --host shard1r1.mongo.cw.com --port 27017 << 'EOF'
rs.initiate()
rs.add('shard1r2.mongo.cw.com:27017')
rs.addArb('router.mongo.cw.com:47018')
EOF
```

Check if everything is right using : `rs.status()`, in the output you should be able to see 3 nodes, with status "PRIMARY", "SECONDARY" & "ARBITER" respectively

##Configure ReplicaSet2 for Shard 2
####Configure Replica Primary on shard2r1.mongo.cw.com

```
mkdir -p /data/shard2 /var/log/mongodb
/opt/mongodb/bin/mongod --replSet shard2 --logpath "/var/log/mongodb/shard2.log" --dbpath /data/shard2 --port 27017 --fork --shardsvr --bind_ip shard2r1.mongo.cw.com
```

####Configure Replica Secondary on shard2r2.mongo.cw.com
 
```
mkdir -p /data/shard2 /var/log/mongodb
/opt/mongodb/bin/mongod --replSet shard2 --logpath "/var/log/mongodb/shard2.log" --dbpath /data/shard2 --port 27017 --fork --shardsvr --bind_ip shard2r2.mongo.cw.com
```

###Configure Replica Arbiter on router.mongo.cw.com

```
mkdir -p /data/shard2/arbiter /var/log/mongodb
/opt/mongodb/bin/mongod --replSet shard2 --logpath "/var/log/mongodb/arbiter_shard2.log" --dbpath /data/shard2/arbiter --port 47019 --fork --bind_ip router.mongo.cw.com
```

####Initialize replica set 2 from mongo shell from replica primary 'shard2r1.mongo.cw.com'

```
/opt/mongodb/bin/mongo --host shard2r1.mongo.cw.com --port 27017 << 'EOF'
rs.initiate()
rs.add('shard2r2.mongo.cw.com:27017')
rs.addArb('router.mongo.cw.com:47019')
EOF
```

##Configure ConfigServer(s) on 'router.mongo.cw.com'

```
mkdir -p /data/config/config-a /data/config/config-b /data/config/config-c 
/opt/mongodb/bin/mongod --logpath "/var/log/mongodb/cfg-a.log" --dbpath /data/config/config-a --port 37019 --fork --configsvr --bind_ip router.mongo.cw.com
/opt/mongodb/bin/mongod --logpath "/var/log/mongodb/cfg-b.log" --dbpath /data/config/config-b --port 37020 --fork --configsvr --bind_ip router.mongo.cw.com
/opt/mongodb/bin/mongod --logpath "/var/log/mongodb/cfg-c.log" --dbpath /data/config/config-c --port 37021 --fork --configsvr --bind_ip router.mongo.cw.com
```

##Configure Mongo Router on 'router.mongo.cw.com'

```
/opt/mongodb/bin/mongos --configdb router.mongo.cw.com:37019,router.mongo.cw.com:37020,router.mongo.cw.com:37021 --fork --logpath "/var/log/mongodb/router.log"
```

##Enable sharding on the shards
###Connect to mongos instance on router.mongo.cw.com and enable sharding

```
/opt/mongodb/bin/mongo --port 27017 <<'EOF'
db.adminCommand( { addshard : "shard1/"+"shard1r1.mongo.cw.com:27017" } );
db.adminCommand( { addshard : "shard2/"+"shard2r1.mongo.cw.com:27017" } );
EOF
```

###Enable sharding for a database 'logs'

```
/opt/mongodb/bin/mongo --port 27017 <<'EOF'
db.adminCommand({enableSharding: "logs"})
EOF
```

###Enable sharding for a collection 'logEvents' using hashed key 'record_id'

```
/opt/mongodb/bin/mongo --port 27017 <<'EOF'
use logs
db.logEvents.ensureIndex( { "record_id": "hashed"} )
sh.shardCollection("logs.logEvents", { "record_id": "hashed" } )
EOF
```

##Verify
Using `sh.status()` from mongos. Output looks something like this:

```
--- Sharding Status ---
  sharding version: {
  "_id" : 1,
  "version" : 3,
  "minCompatibleVersion" : 3,
  "currentVersion" : 4,
  "clusterId" : ObjectId("52884c78701249e73d126128")
}
  shards:
  {  "_id" : "shard1",  "host" : "shard1/ip-10-168-129-36.us-west-1.compute.internal:27017,ip-10-169-41-183:27017" }
  {  "_id" : "shard2",  "host" : "shard2/ip-10-168-10-49.us-west-1.compute.internal:27017,ip-10-168-98-40:27017" }
  databases:
  {  "_id" : "admin",  "partitioned" : false,  "primary" : "config" }
  {  "_id" : "logs",  "partitioned" : true,  "primary" : "shard2" }
    logs.logEvents
      shard key: { "record_id" : "hashed" }
      chunks:
        shard2  2
        shard1  2
      { "record_id" : { "$minKey" : 1 } } -->> { "record_id" : NumberLong("-4611686018427387902") } on : shard2 Timestamp(2, 2)
      { "record_id" : NumberLong("-4611686018427387902") } -->> { "record_id" : NumberLong(0) } on : shard2 Timestamp(2, 3)
      { "record_id" : NumberLong(0) } -->> { "record_id" : NumberLong("4611686018427387902") } on : shard1 Timestamp(2, 4)
      { "record_id" : NumberLong("4611686018427387902") } -->> { "record_id" : { "$maxKey" : 1 } } on : shard1 Timestamp(2, 5)
```

##Sharding with replica set on local machine
port struct
```
Shard Server 1：27020 
Shard Server 2：27021 
Config Server 1 ：27010
Config Server 2 ：27011
Route Server : 27100
```
Create directories
```
mkdir -p /www/mongoDB/shard/{s0,s1,config0,config1,log}
```
##Shard Servers (without replicaSet, not suitable for production env)
```
mongod --fork --shardsvr --port 27020 --dbpath=/www/mongoDB/shard/s0 --logpath /www/mongoDB/shard/log/s0.log

mongod --fork --shardsvr --port 27021 --dbpath=/www/mongoDB/shard/s1 --logpath /www/mongoDB/shard/log/s1.log
```

##Config Server
```
mongod --configsvr --fork --port 27010 --dbpath=/www/mongoDB/shard/config0 --replSet rs1 --logpath /www/mongoDB/shard/log/config.log

mongod --configsvr --fork --port 27011 --dbpath=/www/mongoDB/shard/config1 --replSet rs1 --logpath /www/mongoDB/shard/log/config.log
```
```
mongo --port 27010
> rs.initiate()
rs1:PRIMARY> rs.add("localhost:27011")
> rs.status()
```

##Routing Server
```
mongos --fork --port 27100 --configdb rs1/localhost:27010,localhost:27011 --logpath=/www/mongoDB/shard/log/route.log
```
`--configdb` command line option is used to let the Query router know about the config servers we have setup. You have set up the config server as a standalone mongod process, but as of MongoDB 3.4 that isn't supported: it must be a replicaset.

##Registering the shards with mongos
use shell to connect `mongos`
```
mongo --port 27100
mongos> use config
mongos> db.settings.save( { _id:"chunksize", value: 1 } )
```
In this example, we set the chunk size to its smallest possible size, 1MB for testing. The default chunk size in MongoDB is 64 megabytes. 
```
mongo --port 27100
mongos> use config
mongos> db.settings.save({_id:"chunksize", value: 1})
mongos> sh.addShard("localhost:27020")
mongos> sh.addShard("localhost:27021")
mongos> sh.enableSharding("students")
mongos> use students
mongos> sh.shardCollection("students.grades", {"student_id" : 1})
//Insert lots of data
mongos> for ( i = 1; i < 100000; i++ ) { db.grades.insert({student_id: i, type: "exam", score : Math.random() * 100 })}
mongos> sh.status()
--- Sharding Status ---
  sharding version: {
  	"_id" : 1,
  	"minCompatibleVersion" : 5,
  	"currentVersion" : 6,
  	"clusterId" : ObjectId("5ca0d6198f0ea3b97bc76a72")
  }
  shards:
        {  "_id" : "shard0000",  "host" : "localhost:27020",  "state" : 1 }
        {  "_id" : "shard0001",  "host" : "localhost:27021",  "state" : 1 }
  active mongoses:
        "4.0.8" : 1
  autosplit:
        Currently enabled: yes
  balancer:
        Currently enabled:  yes
        Currently running:  no
        Failed balancer rounds in last 5 attempts:  0
        Migration Results for the last 24 hours:
                6 : Success
  databases:
        {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
                config.system.sessions
                        shard key: { "_id" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                shard0000	1
                        { "_id" : { "$minKey" : 1 } } -->> { "_id" : { "$maxKey" : 1 } } on : shard0000 Timestamp(1, 0)
        {  "_id" : "students",  "primary" : "shard0000",  "partitioned" : true,  "version" : {  "uuid" : UUID("dfc2d66e-1066-469e-9e7f-0a77d55218e7"),  "lastMod" : 1 } }
                students.grades
                        shard key: { "student_id" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                shard0000	7
                                shard0001	6
                        { "student_id" : { "$minKey" : 1 } } -->> { "student_id" : 2 } on : shard0001 Timestamp(4, 1)
                        { "student_id" : 2 } -->> { "student_id" : 14564 } on : shard0000 Timestamp(3, 1)
                        { "student_id" : 14564 } -->> { "student_id" : 21845 } on : shard0000 Timestamp(5, 1)
                        { "student_id" : 21845 } -->> { "student_id" : 31464 } on : shard0000 Timestamp(7, 1)
                        { "student_id" : 31464 } -->> { "student_id" : 38745 } on : shard0001 Timestamp(6, 1)
                        { "student_id" : 38745 } -->> { "student_id" : 47196 } on : shard0001 Timestamp(3, 3)
                        { "student_id" : 47196 } -->> { "student_id" : 54477 } on : shard0000 Timestamp(4, 2)
                        { "student_id" : 54477 } -->> { "student_id" : 62928 } on : shard0000 Timestamp(4, 3)
                        { "student_id" : 62928 } -->> { "student_id" : 70209 } on : shard0001 Timestamp(5, 2)
                        { "student_id" : 70209 } -->> { "student_id" : 78660 } on : shard0001 Timestamp(5, 3)
                        { "student_id" : 78660 } -->> { "student_id" : 85941 } on : shard0000 Timestamp(6, 2)
                        { "student_id" : 85941 } -->> { "student_id" : 94392 } on : shard0000 Timestamp(6, 3)
                        { "student_id" : 94392 } -->> { "student_id" : { "$maxKey" : 1 } } on : shard0001 Timestamp(7, 0)
 
mongos> db.grades.getShardDistribution()

Shard shard0001 at localhost:27021
 data : 2.54MiB docs : 37073 chunks : 6
 estimated data per chunk : 434KiB
 estimated docs per chunk : 6178

Shard shard0000 at localhost:27020
 data : 4.32MiB docs : 62926 chunks : 7
 estimated data per chunk : 632KiB
 estimated docs per chunk : 8989

Totals
 data : 6.86MiB docs : 99999 chunks : 13
 Shard shard0001 contains 37.07% data, 37.07% docs in cluster, avg obj size on shard : 72B
 Shard shard0000 contains 62.92% data, 62.92% docs in cluster, avg obj size on shard : 72B
```
NOTE: The system will need some time to split the chunk to the specified size.

##Convert a Replica Set to a Sharded Cluster

Restart secondary members with the --shardsvr option

Login to primary and run `rs.stepDown()`

Restart the primary with the --shardsvr option

Start the config servers and mongos

Refer:
https://docs.mongodb.com/manual/tutorial/convert-replica-set-to-replicated-shard-cluster/

https://docs.mongodb.com/manual/tutorial/remove-shards-from-cluster/

http://dbversity.com/removeshard/
