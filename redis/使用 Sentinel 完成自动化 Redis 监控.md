# 使用 Sentinel 完成自动化 Redis 监控

Redis-Sentinel 是 Redis 官方推荐的高可用性(HA) 解决方案，当用 Redis 做 Master-slave 的高可用方案时，假如master 宕机了，Redis 本身(包括它的很多客户端) 都没有实现自动进行主备切换，而 Redis-sentinel 本身也是一个独立运行的进程，它能监控多个 master-slave 集群，发现 master 宕机后能进行自懂切换。



它的主要功能有以下几点：

- 不时地监控 redis 是否按照预期良好地运行；
- 如果发现某个 redis 节点运行出现状况，能够通知另外一个进程(例如它的客户端)；
- 能够进行自动切换。当一个 master 节点不可用时，能够选举出 master 的多个 slave (如果有超过一个slave的话) 中的一个来作为新的 master，其它的 slave 节点会将它所追随的 master 的地址改为被提升为 master 的 slave 的新地址。




## 01. 配置 

### 启动 master 与 slave  

在上篇文章中已经做好了 redis 主从配置。由于要学习 Sentinel 的使用，所以还要再多加一个 slave 出来。

```bash
redis-server ./redis.master.conf    # 6379
redis-server ./redis.slave1.conf    # 6380
redis-server ./redis.slave2.conf    # 6381

ps aux | grep redis
yazi        2378  0.1  0.2  29356  2816 ?        Ssl  21:14   0:00 redis-server 127.0.0.1:6379
yazi        2384  0.1  0.3  29356  2872 ?        Ssl  21:14   0:00 redis-server 127.0.0.1:6380
yazi        2389  0.3  0.3  29356  2936 ?        Ssl  21:14   0:00 redis-server 127.0.0.1:6381
yazi        2394  0.0  0.1   4276  1840 pts/0    S+   21:14   0:00 grep --color=auto redis
```

### 配置 Sentinel  

```bash
# 拷贝一份配置文件
yazi@sxyazi:~/redis $ cp /etc/redis/sentinel.conf .
```

文件内容：

```ini
# sentinel 端口
port 26379

# 所要监视的master，"mymaster" 是自定义的项目名称，
# 最后的数字代表同时有N个sentinel程序都判定master挂了，
# 才会去执行主从切换操作的。
sentinel monitor mymaster 127.0.0.1 6379 1

# 检测超时时间(毫秒) 如果超过这个时间还没响应，或响应错误，
# 则是认为master挂了.
sentinel down-after-milliseconds mymaster 30000

# 切换主从时，每次只允许一个 salve 同时同步数据
sentinel parallel-syncs mymaster 1

# 主从切换超时时间(毫秒)，若超过此时间则表示切换失败
sentinel failover-timeout mymaster 180000
```

启动 Sentinel：

```bash
yazi@sxyazi:~/redis $ redis-sentinel ./sentinel.conf
```



## 02. 实验

### 连接上分别看下状态  

```bash
yazi@sxyazi:~ $ redis-cli -p 6379
127.0.0.1:6379> info replication
# Replication
role:master         # 主
connected_slaves:2
slave0:ip=127.0.0.1,port=6380,state=online,offset=22816,lag=0
slave1:ip=127.0.0.1,port=6381,state=online,offset=22816,lag=1


yazi@sxyazi:~ $ redis-cli -p 6380
127.0.0.1:6380> info replication
# Replication
role:slave          # 从
master_host:127.0.0.1
master_port:6379    # 6379端口
master_link_status:up
```

### 把 master 关掉 

```bash
127.0.0.1:6379> shutdown
not connected>
```

### 观察 slave 状态  

```bash
127.0.0.1:6380> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6381    # master 变成了 6381
master_link_status:up
```



## 03. 总结

从上面的试验可以看出，Sentinel 确实起到了作用。当 master(6379) 挂了的时候，6381 自动切换成为了新的 master。

那么为什么是 6381 变成 master 呢？我想让 6380 成为 master 需要怎么做呢？

只需要改一下 slave 的配置文件就行了，把里面的 `slave-priority` 权重选项改的更小一些。

