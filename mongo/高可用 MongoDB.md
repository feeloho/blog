# 高可用 MongoDB

MongoDB 是一个基于分布式文件存储的数据库。由C++语言编写。旨在为WEB应用提供可扩展的高性能数据存储解决方案。

MongoDB 是一个介于关系数据库和非关系数据库之间的产品，是非关系数据库当中功能最丰富，最像关系数据库的。他支持的数据结构非常松散，是类似json的bson格式，因此可以存储比较复杂的数据类型。Mongo最大的特点是他支持的查询语言非常强大，其语法有点类似于面向对象的查询语言，几乎可以实现类似关系数据库单表查询的绝大部分功能，而且还支持对数据建立索引。



## 01. 安装

```bash
apt-get install -y mongodb
root@hostker:~# mongo -version
MongoDB shell version: 2.6.10
```



## 02. 复制集(Replication)

建立复制集

```bash
# 创建测试目录
mkdir -p ~/mongo/17/data
mkdir -p ~/mongo/18/data
mkdir -p ~/mongo/19/data

# 启动3个 mongo 实例, 指定复制集为 rs0
mongod --port 27017 --fork --dbpath ~/mongo/17/data --logpath ~/mongo/17/log --replSet rs0
mongod --port 27018 --fork --dbpath ~/mongo/18/data --logpath ~/mongo/18/log --replSet rs0
mongod --port 27019 --fork --dbpath ~/mongo/19/data --logpath ~/mongo/19/log --replSet rs0
```

连接 27017, 让他成为 primary

```bash
mongo --port 27017

# 初始化后默认是primary
rs.initiate()

# 添加其他实例
rs.add("hostker:27018")
rs.add("hostker:27019")

# 最后可以看下状态
rs.status()
```

测试

```bash
# 27017(primary)
rs0:PRIMARY> use test
rs0:PRIMARY> db.col.insert({a:1})
rs0:PRIMARY> db.col.find()
{ "_id" : ObjectId("594e97394e0d2bf375f8efe3"), "a" : 1 }

# 27018(secondary)
rs0:SECONDARY> show dbs
admin  (empty)
local  1.078GB
test   0.078GB

rs0:SECONDARY> use test

rs0:SECONDARY> db.col.find()
error: { "$err" : "not master and slaveOk=false", "code" : 13435 }
rs0:SECONDARY> rs.slaveOk()  # 设置slaveOk后才能查询数据

rs0:SECONDARY> db.col.find()
{ "_id" : ObjectId("594e97394e0d2bf375f8efe3"), "a" : 1 }
```



## 03. 分片(Sharding)

### 自动分配

```bash
# 创建测试目录
mkdir -p ~/mongo/17/data
mkdir -p ~/mongo/18/data
mkdir -p ~/mongo/19/data
mkdir -p ~/mongo/20

# 启动2台片服务器, 1台config服务器, 1台mongos服务器
mongod --port 27017 --fork --dbpath ~/mongo/17/data --logpath ~/mongo/17/log
mongod --port 27018 --fork --dbpath ~/mongo/18/data --logpath ~/mongo/18/log
mongod --port 27019 --fork --dbpath ~/mongo/19/data --logpath ~/mongo/19/log --configsvr
mongos --port 27020 --fork --configdb 127.0.0.1:27019 --logpath ~/mongo/20/log
```

连接上 mongos 服务器

```bash
mongo --port 27020

# 把刚才启动的片节点加进来
mongos> sh.addShard("127.0.0.1:27017")
mongos> sh.addShard("127.0.0.1:27018")

# 然后查看一下当前的状态
mongos> sh.status()
--- Sharding Status ---
  sharding version: {
	"_id" : 1,
	"version" : 4,
	"minCompatibleVersion" : 4,
	"currentVersion" : 5,
	"clusterId" : ObjectId("594eb27ebdf1f833f0200aae")
}
  shards:
	{  "_id" : "shard0000",  "host" : "127.0.0.1:27017" }
	{  "_id" : "shard0001",  "host" : "127.0.0.1:27018" }
  databases:
	{  "_id" : "admin",  "partitioned" : false,  "primary" : "config" }
```

配置要分片的 库 和 表

```bash
# 设置 test 库分片
sh.enableSharding("test")

# 设置 test 库 col 表按照 _id 字段分片
sh.shardCollection("test.col", {_id: 1})
```

测试

```bash
# 在 mongos 添加数据
mongos> use test
mongos> db.col.insert({a:1})
WriteResult({ "nInserted" : 1 })

# 片1: 
mongo --port 27017
> show dbs
admin  (empty)
local  0.078GB
test   0.078GB

> use test
> db.col.find()
{ "_id" : ObjectId("594ebbfcdf4fba225800a60f"), "a" : 1 }


# 片2: 发现并没有test库, 说明数据只放到了 片1 中
mongo --port 27018
> show dbs
admin  (empty)
local  0.078GB
```

修改默认片大小

```bash
mongos> use config
mongos> db.settings.find()
{ "_id" : "chunksize", "value" : 64 }   # 默认 64M

db.settings.update({_id: "chunksize"}, {$set: {value: 1}})
```

插入数据

```bash
# 插入数据之前: 
mongos> sh.status()
--- Sharding Status ---
  sharding version: {
	"_id" : 1,
	"version" : 4,
	"minCompatibleVersion" : 4,
	"currentVersion" : 5,
	"clusterId" : ObjectId("594eb27ebdf1f833f0200aae")
}
  shards:
	{  "_id" : "shard0000",  "host" : "127.0.0.1:27017" }
	{  "_id" : "shard0001",  "host" : "127.0.0.1:27018" }
  databases:
	{  "_id" : "admin",  "partitioned" : false,  "primary" : "config" }
	{  "_id" : "test",  "partitioned" : true,  "primary" : "shard0000" }
		test.col
			shard key: { "_id" : 1 }
			chunks:
				shard0000	1
			{ "_id" : { "$minKey" : 1 } } -->> { "_id" : { "$maxKey" : 1 } } on : shard0000 Timestamp(1, 0)
	{  "_id" : "setting",  "partitioned" : false,  "primary" : "shard0001" }


# 添加数据
for(var i=1; i<=50000; i++) db.col.insert({_id: i, val: "yaziaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"})


# 插入 5W 条数据后: 
mongos> sh.status()
--- Sharding Status ---
  sharding version: {
	"_id" : 1,
	"version" : 4,
	"minCompatibleVersion" : 4,
	"currentVersion" : 5,
	"clusterId" : ObjectId("594eb27ebdf1f833f0200aae")
}
  shards:
	{  "_id" : "shard0000",  "host" : "127.0.0.1:27017" }
	{  "_id" : "shard0001",  "host" : "127.0.0.1:27018" }
  databases:
	{  "_id" : "admin",  "partitioned" : false,  "primary" : "config" }
	{  "_id" : "test",  "partitioned" : true,  "primary" : "shard0000" }
		test.col
			shard key: { "_id" : 1 }
			chunks:
				shard0001	8
				shard0000	8
			{ "_id" : { "$minKey" : 1 } } -->> { "_id" : 0 } on : shard0001 Timestamp(2, 0)
			{ "_id" : 0 } -->> { "_id" : 5761 } on : shard0001 Timestamp(4, 0)
			{ "_id" : 5761 } -->> { "_id" : 10280 } on : shard0001 Timestamp(5, 0)
			{ "_id" : 10280 } -->> { "_id" : 14051 } on : shard0001 Timestamp(6, 0)
			{ "_id" : 14051 } -->> { "_id" : 17500 } on : shard0001 Timestamp(7, 0)
			{ "_id" : 17500 } -->> { "_id" : 20756 } on : shard0001 Timestamp(8, 0)
			{ "_id" : 20756 } -->> { "_id" : 23840 } on : shard0001 Timestamp(9, 0)
			{ "_id" : 23840 } -->> { "_id" : 26752 } on : shard0000 Timestamp(9, 1)
			{ "_id" : 26752 } -->> { "_id" : 29585 } on : shard0000 Timestamp(6, 2)
			{ "_id" : 29585 } -->> { "_id" : 32403 } on : shard0000 Timestamp(6, 4)
			{ "_id" : 32403 } -->> { "_id" : 35162 } on : shard0000 Timestamp(6, 6)
			{ "_id" : 35162 } -->> { "_id" : 37892 } on : shard0000 Timestamp(6, 8)
			{ "_id" : 37892 } -->> { "_id" : 40566 } on : shard0000 Timestamp(6, 10)
			{ "_id" : 40566 } -->> { "_id" : 43213 } on : shard0000 Timestamp(7, 2)
			{ "_id" : 43213 } -->> { "_id" : ObjectId("594ebbfcdf4fba225800a60f") } on : shard0000 Timestamp(7, 3)
			{ "_id" : ObjectId("594ebbfcdf4fba225800a60f") } -->> { "_id" : { "$maxKey" : 1 } } on : shard0001 Timestamp(3, 0)
	{  "_id" : "setting",  "partitioned" : false,  "primary" : "shard0001" }

```



### 预先分配数据

```bash
# 查看片状态
mongos> sh.status()
--- Sharding Status ---
  sharding version: { "_id" : 1, "version" : 3 }
  shards:
	{  "_id" : "shard0000",  "host" : "127.0.0.1:27017" }
	{  "_id" : "shard0001",  "host" : "127.0.0.1:27018" }
  databases:
	{  "_id" : "admin",  "partitioned" : false,  "primary" : "config" }
	{  "_id" : "test",  "partitioned" : true,  "primary" : "shard0000" }
		test.col chunks:
				shard0000	1
			{ "_id" : { $minKey : 1 } } -->> { "_id" : { $maxKey : 1 } } on : shard0000 { "t" : 1000, "i" : 0 }

# 添加10条记录
for(var i=1; i<=10; i++) sh.splitAt("test.col", {_id: i*10})

# 再次查看状态, 已经均匀的散落在2个片节点上
mongos> sh.status()
--- Sharding Status ---
  sharding version: { "_id" : 1, "version" : 3 }
  shards:
	{  "_id" : "shard0000",  "host" : "127.0.0.1:27017" }
	{  "_id" : "shard0001",  "host" : "127.0.0.1:27018" }
  databases:
	{  "_id" : "admin",  "partitioned" : false,  "primary" : "config" }
	{  "_id" : "test",  "partitioned" : true,  "primary" : "shard0000" }
		test.col chunks:
				shard0001	5
				shard0000	5
			{ "_id" : { $minKey : 1 } } -->> { "_id" : 10 } on : shard0001 { "t" : 2000, "i" : 0 }
			{ "_id" : 10 } -->> { "_id" : 20 } on : shard0001 { "t" : 3000, "i" : 0 }
			{ "_id" : 20 } -->> { "_id" : 30 } on : shard0001 { "t" : 4000, "i" : 0 }
			{ "_id" : 30 } -->> { "_id" : 40 } on : shard0001 { "t" : 5000, "i" : 0 }
			{ "_id" : 40 } -->> { "_id" : 50 } on : shard0001 { "t" : 6000, "i" : 0 }
			{ "_id" : 50 } -->> { "_id" : 60 } on : shard0000 { "t" : 6000, "i" : 1 }
			{ "_id" : 60 } -->> { "_id" : 70 } on : shard0000 { "t" : 1000, "i" : 13 }
			{ "_id" : 70 } -->> { "_id" : 80 } on : shard0000 { "t" : 1000, "i" : 15 }
			{ "_id" : 80 } -->> { "_id" : 100 } on : shard0000 { "t" : 2000, "i" : 2 }
			{ "_id" : 100 } -->> { "_id" : { $maxKey : 1 } } on : shard0000 { "t" : 2000, "i" : 3 }
```

