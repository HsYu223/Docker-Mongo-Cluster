# Docker-MongoCluster

## 2023/07/01 更新6.0.6版本

預計測試Mongo Cluster架構如下圖：

![測試架構](https://raw.githubusercontent.com/HsYu223/Docker-Mongo-Cluster/master/images/Mongo%20Cluster.png)

1 建立Container volume

```bash
docker volume create mongo1data
docker volume create mongo2data
docker volume create mongo3data
docker volume create mongo4data
docker volume create mongo5data
docker volume create mongo6data
```

-------------------------------------------------------------------------------------------------------

2 建立第一個Replica Set (mongo-set1)

```bash
docker run -p 27017:27017 --name mongo1 -d -v mongo1data:/data/db mongo:6.0.6 mongod --shardsvr --replSet mongo-set1 --port 27017

docker run -p 27018:27018 --name mongo2 -d -v mongo2data:/data/db mongo:6.0.6 mongod --shardsvr --replSet mongo-set1 --port 27018

docker run -p 27019:27019 --name mongo3 -d -v mongo3data:/data/db mongo:6.0.6 mongod --shardsvr --replSet mongo-set1 --port 27019

docker exec -it mongo1 mongosh {mongo1 ip}:27017
	config = { "_id": "mongo-set1", "members": [{"_id":0,"host":"{mongo1 ip}:27017"},{"_id":1,"host":"{mongo2 ip}::27018"},{"_id":2,"host":"{mongo3 ip}::27019",priority:0,hidden:true}] }
	rs.initiate(config)
	rs.status()
```

-------------------------------------------------------------------------------------------------------
3 建立第二個Replica Set (mongo-set2)

```bash
docker run -p 27027:27027 --name mongo4 -d -v mongo4data:/data/db mongo:6.0.6 mongod --shardsvr --replSet mongo-set2 --port 27027

docker run -p 27028:27028 --name mongo5 -d -v mongo5data:/data/db mongo:6.0.6 mongod --shardsvr --replSet mongo-set2 --port 27028

docker run -p 27029:27029 --name mongo6 -d -v mongo6data:/data/db mongo:6.0.6 mongod --shardsvr --replSet mongo-set2 --port 27029

docker exec -it mongo1 mongosh {mongo4 ip}:27027
	config = { "_id": "mongo-set2", "members": [{"_id":0,"host":"{mongo4 ip}:27027"},{"_id":1,"host":"{mongo5 ip}:27028"},{"_id":2,"host":"{mongo6 ip}:27029",priority:0,hidden:true}] }
	rs.initiate(config)
	rs.status()
```

-------------------------------------------------------------------------------------------------------
4 建立一組Config Servers

```bash
docker run -p 47017:47017 --name mongo-cfg1 -d mongo:6.0.6 /bin/bash -c " mkdir -p /tmp/mongo/db; mongod --configsvr --replSet config-set --dbpath /tmp/mongo/db --port 47017 --bind_ip 0.0.0.0 "

docker run -p 47018:47018 --name mongo-cfg2 -d mongo:6.0.6 /bin/bash -c " mkdir -p /tmp/mongo/db; mongod --configsvr --replSet config-set --dbpath /tmp/mongo/db --port 47018 --bind_ip 0.0.0.0 "

docker exec -it mongo-cfg1 mongosh {mongo-cfg1 ip}:47017
	rs.initiate({"_id":"config-set","configsvr":true,"members":[{"_id":0,"host":"{mongo-cfg1 ip}:47017"},{"_id":1,"host":"{mongo-cfg2 ip}:47018"}]})
```

-------------------------------------------------------------------------------------------------------
5 建立mongos (Router Service)

```bash
docker run -p 37017:37017 --name mongo-s1 --net host -d mongo /bin/bash -c " mongos --configdb config-set/SRVDOCKER-T:47017,SRVDOCKER-T:47018 --port 37017 "
docker run -p 37018:37018 --name mongo-s2 --net host -d mongo /bin/bash -c " mongos --configdb config-set/SRVDOCKER-T:47017,SRVDOCKER-T:47018 --port 37018 "

docker run -p 37017:37017 --name mongo-s1 -d mongo:6.0.6 /bin/bash -c " mongos --configdb config-set/{mongo-cfg1 ip}:47017,{mongo-cfg2 ip}:47018 --port 37017 "
docker run -p 37017:37018 --name mongo-s2 -d mongo:6.0.6 /bin/bash -c " mongos --configdb config-set/{mongo-cfg1 ip}:47017,{mongo-cfg2 ip}:47018 --port 37018 "

docker exec -it mongo-s1 mongosh {mongo-s1 ip}:37017
	sh.addShard("mongo-set1/{mongo1 ip}:27017,{mongo2 ip}:27018")
	sh.addShard("mongo-set2/{mongo4 ip}:27027,{mongo5 ip}:27028")
```

6. 延續從mongos 建立分片索引

```bash
sh.enableSharding('AccessLog')
db.AppEvent.ensureIndex( { _id : "hashed" } )
sh.shardCollection("AccessLog.AppEvent", { "_id": "hashed" } )

use AccessLog;
var objs = []; for (var i=0;i<100;i++){	objs.push({"serialNo":i,"name":"user"+i}); }
db.AppEvent.insert(objs);
```

-------------------------------------------------------------------------------------------------------
6 MongoDB Tools

[Robo 3T](https://robomongo.org/)

![Robo 3T](images/Robo 3T.png)

[MongoDB Compass](https://docs.mongodb.com/compass/current/)

![MongoDB Compass](images/MongoDB Compass.png)

-------------------------------------------------------------------------------------------------------
7 Docker Volume 備份&還原

```bash
// volume 備份
docker run --rm --volumes-from mongo3 -v $(pwd):/backup busybox tar cvf /backup/mongo3data.tar /data

//volume 還原
docker run --rm --volumes-from mongo3 -v $(pwd):/backup busybox tar xvf /backup/mongo3data.tar
```

-------------------------------------------------------------------------------------------------------
8 為正式機加入使用者帳密

產生金鑰設定(每台測試機須保持金鑰一致性)

```bash
openssl rand -base64 756 > /data/mongo-keyfile
sudo chmod 400 /data/mongo-keyfile
sudo chown 999 /data/mongo-keyfile
```

Create replSet 1

```bash
sudo docker run -v /data/mongo-keyfile:/data/mongo-keyfile -p 27018:27018 --name repl1-mongo1 --net host --restart always -d -v /data/repl1-mongo1-data:/data/db mongo mongod --shardsvr --replSet repl-set1 --port 27018 --auth --keyFile /data/mongo-keyfile

sudo docker run -v /data/mongo-keyfile:/data/mongo-keyfile -p 27019:27019 --name repl1-mongo2 --net host --restart always -d -v /data/repl1-mongo2-data:/data/db mongo mongod --shardsvr --replSet repl-set1 --port 27019 --auth --keyFile /data/mongo-keyfile

Arbiter	
sudo docker run -v /data/mongo-keyfile:/data/mongo-keyfile -p 27020:27020 --name repl1-arb --net host --restart always -d -v /data/repl1-arb-data:/data/arb mongo /bin/bash -c " mkdir -p /data/arb | mongod --replSet repl-set1 --dbpath /data/arb --port 27020 --auth --keyFile /data/mongo-keyfile"

go to srvmongoDB2
sudo docker exec -it repl1-mongo1 mongo localhost:27018
	rs.initiate()
	use admin
	db.createUser(
	  {
		user: "admin",
		pwd: "1q2w3e4r5t_",
		roles: [
		  { role: "root", db: "admin" }
		]
	  }
	);
	rs.status();
	db.auth("admin","1q2w3e4r5t_");
	rs.add("srvmongoDB1:27019")
	rs.addArb("srvmongoDB3:27020")
	rs.status();
```	

Create replSet 2

```bash
sudo docker run -v /data/mongo-keyfile:/data/mongo-keyfile -p 27018:27018 --name repl2-mongo1 --net host --restart always -d -v /data/repl2-mongo1-data:/data/db mongo mongod --shardsvr --replSet repl-set2 --port 27018 --auth --keyFile /data/mongo-keyfile

sudo docker run -v /data/mongo-keyfile:/data/mongo-keyfile -p 27019:27019 --name repl2-mongo2 --net host --restart always -d -v /data/repl2-mongo2-data:/data/db mongo mongod --shardsvr --replSet repl-set2 --port 27019 --auth --keyFile /data/mongo-keyfile

Arbiter	
sudo docker run -v /data/mongo-keyfile:/data/mongo-keyfile -p 27020:27020 --name repl2-arb --net host --restart always -d -v /data/repl2-arb-data:/data/arb mongo /bin/bash -c " mkdir -p /data/arb | mongod --replSet repl-set2 --dbpath /data/arb --port 27020 --auth --keyFile /data/mongo-keyfile "

go to srvmongoDB3
sudo docker exec -it repl2-mongo1 mongo localhost:27018
	rs.initiate()
	use admin
	db.createUser(
	  {
		user: "admin",
		pwd: "1q2w3e4r5t_",
		roles: [
		  { role: "root", db: "admin" }
		]
	  }
	);
	rs.status();
	db.auth("admin","1q2w3e4r5t_");
	rs.add("srvmongoDB2:27019")
	rs.addArb("srvmongoDB1:27020")
	rs.status();
```

Create config server

```bash
sudo docker run -v /data/mongo-keyfile:/data/mongo-keyfile -p 47018:47018 --name mongo-cfg1 --net host --restart always -d mongo /bin/bash -c " mkdir -p /tmp/mongo/db; mongod --configsvr --replSet config-set --port 47018 --dbpath /tmp/mongo/db --keyFile /data/mongo-keyfile"

sudo docker run -v /data/mongo-keyfile:/data/mongo-keyfile -p 47019:47019 --name mongo-cfg2 --net host --restart always -d mongo /bin/bash -c " mkdir -p /tmp/mongo/db; mongod --configsvr --replSet config-set --port 47019 --dbpath /tmp/mongo/db --keyFile /data/mongo-keyfile"

sudo docker run -v /data/mongo-keyfile:/data/mongo-keyfile -p 47020:47020 --name mongo-cfg3 --net host --restart always -d mongo /bin/bash -c " mkdir -p /tmp/mongo/db; mongod --configsvr --replSet config-set --port 47020 --dbpath /tmp/mongo/db --keyFile /data/mongo-keyfile"

sudo docker exec -it mongo-cfg1 mongo localhost:47018
	rs.initiate()
	use admin
	db.createUser(
	  {
		user: "admin",
		pwd: "1q2w3e4r5t_",
		roles: [
		  { role: "root", db: "admin" }
		]
	  }
	);
	rs.status();
	db.auth("admin","1q2w3e4r5t_");
	rs.add("srvmongoDB2:47019")
	rs.add("srvmongoDB1:47020")
	rs.status();
```

Create mongos

```bash
sudo docker run -v /data/mongo-keyfile:/data/mongo-keyfile -p 37018:37018 --name mongo-s1 --net host --restart always -d mongo /bin/bash -c " mongos --configdb config-set/srvmongoDB3:47018,srvmongoDB2:47019,srvmongoDB1:47020 --port 37018 --keyFile /data/mongo-keyfile"

sudo docker run -v /data/mongo-keyfile:/data/mongo-keyfile -p 37019:37019 --name mongo-s2 --net host --restart always -d mongo /bin/bash -c " mongos --configdb config-set/srvmongoDB3:47018,srvmongoDB2:47019,srvmongoDB1:47020 --port 37019 --keyFile /data/mongo-keyfile"

sudo docker run -v /data/mongo-keyfile:/data/mongo-keyfile -p 37020:37020 --name mongo-s3 --net host --restart always -d mongo /bin/bash -c " mongos --configdb config-set/srvmongoDB3:47018,srvmongoDB2:47019,srvmongoDB1:47020 --port 37020 --keyFile /data/mongo-keyfile"

sudo docker exec -it mongo-s1 mongo srvmongoDB1:37018
	use admin;
	db.auth("admin","1q2w3e4r5t_");
	sh.addShard("repl-set1/srvmongoDB1:27019")
	sh.addShard("repl-set2/srvmongoDB2:27019")
	
sudo docker exec -it mongo-s2 mongo srvmongoDB2:37019
	use admin;
	db.auth("admin","1q2w3e4r5t_");
	sh.addShard("repl-set1/srvmongoDB1:27019")
	sh.addShard("repl-set2/srvmongoDB2:27019")
	
sudo docker exec -it mongo-s3 mongo srvmongoDB3:37020
	use admin;
	db.auth("admin","1q2w3e4r5t_");
	sh.addShard("repl-set1/srvmongoDB1:27019")
	sh.addShard("repl-set2/srvmongoDB2:27019")
```
	
```bash
use AccessLog;

sh.enableSharding('AccessLog')

db.accesslog.ensureIndex( { _id : "hashed" } )

sh.shardCollection("AccessLog.accesslog", { "_id": "hashed" } )
```
