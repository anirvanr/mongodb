# Mongo replica:
Replication is referred to the process of ensuring that the same data is available on more than one Mongo DB Server.
A replica set is a group of mongod instances that host the same data set. 
Replica set can have only one primary node.
All write operations go to primary.
At the time of automatic failover or maintenance, election establishes for primary and a new primary node is elected.
After the recovery of failed node, it again join the replica set and works as a secondary node.
When a primary does not communicate with the other members of the set for more than the configured period (10 seconds by default), an eligible secondary calls for an election to nominate itself as the new primary.
Replica set members send heartbeats (pings) to each other every two seconds. If a heartbeat does not return within 10 seconds, the other members mark the delinquent member as inaccessible.

Replication setup:
Create a `/etc/yum.repos.d/mongodb-org-4.0.repo` file
```
[mongodb-org-4.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/4.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-4.0.asc
```
```
yum install -y mongodb-org
yum -y install openssl openssl-devel perl vim
```
Setup 4 new instances (1 primary, 2 replica, 1 arbiter):
```
mkdir -p /srv/mongodb/rs0-0  /srv/mongodb/rs0-1 /srv/mongodb/rs0-2 /srv/mongodb/rs0-arb
```
This will create directories called "rs0-0", "rs0-1", "rs0-2" and "rs0-arb", which will contain the instances’ database files.
```
mongod --fork --logpath /var/log/mongod.log --replSet rs0 --port 27017 --bind_ip localhost,127.0.0.1 --dbpath /srv/mongodb/rs0-0 --smallfiles --oplogSize 128

mongod --fork --logpath /var/log/mongod.log --replSet rs0 --port 27018 --bind_ip localhost,127.0.0.1 --dbpath /srv/mongodb/rs0-1 --smallfiles --oplogSize 128

mongod --fork --logpath /var/log/mongod.log --replSet rs0 --port 27019 --bind_ip localhost,127.0.0.1 --dbpath /srv/mongodb/rs0-2 --smallfiles --oplogSize 128

mongod --fork --logpath /var/log/mongod.log --replSet rs0 --port 27020 --bind_ip localhost,127.0.0.1 --dbpath /srv/mongodb/rs0-arb
```
This starts each instance as a member of a replica set named `rs0`, each running on a distinct port, and specifies the path to your data directory with the `--dbpath` setting. The `--smallfiles` and `--oplogSize` settings reduce the disk space that each mongod instance uses.

The Oplog (operations log) is a special capped collection that keeps a rolling record of all operations that modify (updates, deletes, inserts) the data stored in your databases. When ever client writes or reads from primary node the primary node logs it in its oplog. Now the two secondary nodes replicate the oplog’s operations and in-turn has the same data as primary(asynchronous process). All replica set members contain a copy of the oplog, in the `local.oplog.rs` collection, which allows them to maintain the current state of the database.The MongoDB Oplog is created after starting a replica set member for the first time with a default size. By default 5% of the partition where the dbpath lives is reserved for the oplog.
You can tail the oplog
```
use local
db.getCollection('oplog.rs').find()
```
To see the replication status, run `db.printSlaveReplicationInfo()` command on primary. This will show you if the secondaries are up to date or lagging.
`rs.printReplicationInfo()` shows the configured/default oplog size, which the replication cluster is using currently. The date range shows the start and end date of the MongoDB operations stored in the oplogs.
You can demote a primary to a secondary using the stepDown function `rs.stepDown()`.When you start a replica set member for the first time, MongoDB creates an oplog of a default size if you do not specify the oplog size.
For Unix and Windows systems the default oplog size depends on the storage engine:

| Storage Engine            | Default Oplog Size    | Lower Bound | Upper Bound |
|---------------------------|-----------------------|-------------|-------------|
| In-Memory Storage Engine  | 5% of physical memory | 50 MB       | 50 GB       |
| WiredTiger Storage Engine | 5% of free disk space | 50 MB       | 50 GB       |
| MMAPv1 Storage Engine     | 5% of free disk space | 50 MB       | 50 GB       |

Connect to one of your mongod instances through the mongo shell.
```
mongo --port 27017
```
In the mongo shell, use `rs.initiate()` to initiate the replica set.
Display the current replica configuration by issuing the following command: `rs.conf()`
Now we have the one server added in replica, lets add the rest of servers.
```
rs.add("localhost:27018")
rs.add("localhost:27019")
rs.addArb("localhost:27020")
```
You can enter the `rs.status()` in the shell prompt to check how many servers are in replica set. All members should have a `health` value of `1`. 
```
rs0:PRIMARY> rs.status()
{
	"set" : "rs0",
	"date" : ISODate("2019-03-30T07:02:33.563Z"),
	"myState" : 1,
	"term" : NumberLong(1),
	"syncingTo" : "",
	"syncSourceHost" : "",
	"syncSourceId" : -1,
	"heartbeatIntervalMillis" : NumberLong(2000),
	"optimes" : {
		"lastCommittedOpTime" : {
			"ts" : Timestamp(1553929346, 1),
			"t" : NumberLong(1)
		},
		"readConcernMajorityOpTime" : {
			"ts" : Timestamp(1553929346, 1),
			"t" : NumberLong(1)
		},
		"appliedOpTime" : {
			"ts" : Timestamp(1553929346, 1),
			"t" : NumberLong(1)
		},
		"durableOpTime" : {
			"ts" : Timestamp(1553929346, 1),
			"t" : NumberLong(1)
		}
	},
	"lastStableCheckpointTimestamp" : Timestamp(1553929320, 1),
	"members" : [
		{
			"_id" : 0,
			"name" : "localhost:27017",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"uptime" : 93,
			"optime" : {
				"ts" : Timestamp(1553929346, 1),
				"t" : NumberLong(1)
			},
			"optimeDate" : ISODate("2019-03-30T07:02:26Z"),
			"syncingTo" : "",
			"syncSourceHost" : "",
			"syncSourceId" : -1,
			"infoMessage" : "could not find member to sync from",
			"electionTime" : Timestamp(1553929318, 2),
			"electionDate" : ISODate("2019-03-30T07:01:58Z"),
			"configVersion" : 4,
			"self" : true,
			"lastHeartbeatMessage" : ""
		},
		{
			"_id" : 1,
			"name" : "localhost:27018",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 22,
			"optime" : {
				"ts" : Timestamp(1553929346, 1),
				"t" : NumberLong(1)
			},
			"optimeDurable" : {
				"ts" : Timestamp(1553929346, 1),
				"t" : NumberLong(1)
			},
			"optimeDate" : ISODate("2019-03-30T07:02:26Z"),
			"optimeDurableDate" : ISODate("2019-03-30T07:02:26Z"),
			"lastHeartbeat" : ISODate("2019-03-30T07:02:32.417Z"),
			"lastHeartbeatRecv" : ISODate("2019-03-30T07:02:33.433Z"),
			"pingMs" : NumberLong(0),
			"lastHeartbeatMessage" : "",
			"syncingTo" : "localhost:27017",
			"syncSourceHost" : "localhost:27017",
			"syncSourceId" : 0,
			"infoMessage" : "",
			"configVersion" : 4
		},
		{
			"_id" : 2,
			"name" : "localhost:27019",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 14,
			"optime" : {
				"ts" : Timestamp(1553929346, 1),
				"t" : NumberLong(1)
			},
			"optimeDurable" : {
				"ts" : Timestamp(1553929346, 1),
				"t" : NumberLong(1)
			},
			"optimeDate" : ISODate("2019-03-30T07:02:26Z"),
			"optimeDurableDate" : ISODate("2019-03-30T07:02:26Z"),
			"lastHeartbeat" : ISODate("2019-03-30T07:02:32.417Z"),
			"lastHeartbeatRecv" : ISODate("2019-03-30T07:02:33.432Z"),
			"pingMs" : NumberLong(0),
			"lastHeartbeatMessage" : "",
			"syncingTo" : "localhost:27017",
			"syncSourceHost" : "localhost:27017",
			"syncSourceId" : 0,
			"infoMessage" : "",
			"configVersion" : 4
		},
		{
			"_id" : 3,
			"name" : "localhost:27020",
			"health" : 1,
			"state" : 7,
			"stateStr" : "ARBITER",
			"uptime" : 7,
			"lastHeartbeat" : ISODate("2019-03-30T07:02:32.420Z"),
			"lastHeartbeatRecv" : ISODate("2019-03-30T07:02:32.440Z"),
			"pingMs" : NumberLong(0),
			"lastHeartbeatMessage" : "",
			"syncingTo" : "",
			"syncSourceHost" : "",
			"syncSourceId" : -1,
			"infoMessage" : "",
			"configVersion" : 4
		}
	],
	"ok" : 1,
	"operationTime" : Timestamp(1553929346, 1),
	"$clusterTime" : {
		"clusterTime" : Timestamp(1553929346, 1),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	}
}
```
State gives more information about the specific state that member is in:
```
    0 = Starting up (phase 1)
    1 = Primary
    2 = Secondary
    3 = Recovering
    4 = Fatal error
    5 = Starting up (phase 2)
    6 = Unknown state
    7 = Arbiter
    8 = Down
```
The `optimeDate` allows you to see whether a member is behind on the replication sync. The timestamp is the last applied log item so if it’s up to date, it’ll be very close to the current actual time on the server. Only the node with the latest optime (the timestamp of the most recent oplog record) can be elected as the primary node. 

Test Replication:-
Connect to the primary member
```
mongo --port 27017
```
execute
```
use exampleDB
for (var i = 0; i <= 10; i++) db.exampleCollection.insert( { x : i } )
```
If your replica set is configured properly, the data should be present on your secondary members as well as the primary.
Connect one of your secondary members 
```
mongo --port 27018
``` 
run: `rs.slaveOk()`

By default, read queries are not allowed on secondary members, above command enable it.
```
use exampleDB
db.exampleCollection.find()
```
Do not execute the command "rs.initiate()" in all the servers, Because this command will make the server as primary by default.`rs.stepDown()` is a good way to step down the primary.If you need to shut down a secondary, connect to it, make sure you are on the admin database `use admin`, and run `db.shutdownServer()`.When a primary does not communicate with the other members of the set for more than 10 seconds, the replica set will attempt to select another member to become the new primary.To remove a server from the configuration set, we need to use the `rs.remove("127.0.0.1:27019")` command but first perform a shutdown of the instance `db.shutdownServer()` which you want to remove.
IMPORTANT: Elections are essential for independent operation of a replica set; however, elections take time to complete. While an election is in process, the replica set has no primary and cannot accept writes. MongoDB avoids elections unless necessary.

`Arbiter` nodes only participate in voting and cannot be elected as the primary node or sync data from the primary node. If a replica set has an even number of members then you can add an arbiter. 
An arbiter will always be an arbiter.You cannot reconfigure an arbiter to become a non-arbiter, or vice versa.
Do not run the arbiter on the same system as a member of the replica set.
Because a replica set can have up to 50 members, but only 7 voting members, non-voting members allow a replica set to have more than seven members.
Non-voting members must have priority of 0.
The value of priority can be any floating point number between 0 and 1000.
Specify higher priority values to make a member more eligible to become primary
Default: 1 for primary/secondary; 0 for arbiters.
Members with 0 priority can never become primary. These members are called passive members.
When the primary node finds there is a secondary node with a higher priority, and the data latency on the secondary node is within 10 seconds, the primary node will perform an active stepDown and make the secondary node with a higher priority eligible for being the primary node. 

To configure a delayed secondary member, set its priority value to 0, its hidden value to true, and its slaveDelay value to the number of seconds to delay.
```
cfg = rs.conf()
cfg.members[0].priority = 0
cfg.members[0].hidden = true
cfg.members[0].slaveDelay = 1200
rs.reconfig(cfg)
```

A `hidden member` maintains a copy of the primary’s data set but is invisible to client applications.Hidden members do vote in elections. They must always be priority 0 members and so they cannot become primary.

`Slave Delay`: It’s always possible for your data to be nuked by human error: someone might accidentally drop your main database or a newly deployed version of your application might have a bug that replaces all of your data with garbage. To defend against that type of problem, you can set up a delayed secondary using the slaveDelay setting.
```
{
   "_id" : <num>,
   "host" : <hostname:port>,
   "priority" : 0,
   "slaveDelay" : <seconds>,
   "hidden" : true
}
```
Delayed members:
  •	Must be priority 0 members. Set the priority to 0 to prevent a delayed member from becoming primary.
  •	Should be hidden members. Always prevent applications from seeing and querying delayed members.
  •	Has vote 1 by default
  •	Mainly used for backup
  
`rs.isMaster()` command to check whether the current member is master or not

In the election few of the members participate and give votes to determine primary member. But there are also a few members, who do not participate in voting. These members are called `non-voting members`.To configure a member as non-voting, set its votes value to 0.

How to check secondary is synced now or not:
You can use output of rs.status(). If secondary is synced and wasn't created with slaveDelay option then optime and optimeDate of secondary should be equal or close (if there are current operations) to those of primary. In that case stateStr should be equal to SECONDARY.
If secondary has been created with slaveDelay then optime and optimeDate can be different but stateStr and lastHeartbeatMessage will indicate if there is some lag.

The voting is done by a `majority of voting members`(over 50%).

Imagine a Replica Set with three (voting) members. Let's say that Node A is primary, and nodes B+C are secondaries. Node A goes down, so nodes B+C go to election. They still do form a majority (two out of three ie 66%). The election is first decided by priority. If both Nodes B & C have the same priority, then the one who is most up to date in respect to the failed primary (oplog) wins. Let's say it's Node B. Once node A comes back alive, there is no new election. Node B remains the master, and C+A are now secondaries. You can change this by setting the priority of the servers. So if the old Primary Node A comes back online it will become the Primary again.
```
var conf = rs.config()
```
The Primary is the first in the members array.
```
conf.members[0].priority = 10
```
Apply the new configuration. This needs to be done from the current Primary.
```
rs.reconfig(conf)
```
On the other hand, if two nodes go down you don't have a majority, so the replica set can't accept updates (apply writes) any more until at least one of the two failing servers becomes alive (and connected by the single surviving node) again.

Imagine now a Replica Set with four (voting) members. Let's say that Node A is primary, and nodes B+C+D are secondaries. Node A goes down, so nodes B+C+D go to election. They of course form majority (three out of four)

However, if two nodes go down you don't have a majority (two out of four), so the replica set is again at read only mode.

So that's why an odd number is recommended; If you loose a single member in a 3 members replica set, it's the same as loosing a single member in a 4 members replica set: you still gain quorum majority and a new primary can be elected (the RS can still elect a new master by majority). On the other hand, if you loose two members in a 3 members replica set or a 4 members replica set (or n/2 members of n-members replica set) - again - the impact is the same: No new leader can be voted by election.

So, to make a long story short, there is no redundancy gain by having an even number of members in a replica set.

There are times when you need to do work on the primary server in a replica set. To do this you need to take the server out of the replica set temporarly. To do this you can use the `rs.stepDown(seconds)` command. For example, to step down a server for 3 minutes:
```
rs.stepDown(60*3)
```

Sometimes you may need to work on a Secondary server and you need to ensure that it does not become a Primary while you are working on it. To do this you can issue a `rs.freeze(seconds)` to temporarly stop the Secondary from becoming a Primary.
