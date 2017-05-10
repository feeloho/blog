# Redis 主从配置

前面写了关于 MySQL 主从配置的文章，这篇文章来写一下 Redis 的主从配置。相对于 MySQL 来说简单了很多。



## 01. 安装

```bash
sudo apt-get update
sudo apt-get install redis-server
mkdir /tmp/redis  # 创建测试目录
```



## 02. 拷贝两份配置

```bash
yazi@sxyazi:~/redis $ cp /etc/redis/redis.conf ./redis.master.conf
yazi@sxyazi:~/redis $ cp /etc/redis/redis.conf ./redis.slave.conf
```



## 03. 分别修改配置文件

**redis.master.conf**: 

```ini
daemonize yes                      # 守护进程
pidfile /var/run/redis_master.pid  # PID文件
dir /tmp/redis/                    # 数据存储目录

port 6379                          # 监听端口
bind 127.0.0.1                     # 监听地址，若要允许外网访问填写对应IP
# requirepass abcd1234             # 密码

appendonly yes                     # 开启AOF
appendfilename master.aof          # AOF文件名
# appendfsync no                   # AOF写入方式，交由操作系统
# appendfsync always               # 每执行命令就写一次
appendfsync everysec               # 每秒写一次

dbfilename master.rdb              # RDB文件名
auto-aof-rewrite-percentage 100    # AOF自动重写增长率
auto-aof-rewrite-min-size 64mb     # AOF自动重写最小大小
```

**redis.salve.conf**: 

```ini
daemonize yes                     # 守护进程
pidfile /var/run/redis_slave.pid  # PID文件

port 6380                         # 监听端口
bind 127.0.0.1                    # 监听地址，若要允许外网访问填写对应IP

slaveof 127.0.0.1 6379            # 作为 127.0.0.1:6379 的Slave
# masterauth abcd1234             # 密码认证

dir /tmp/redis/                   # 数据存储目录
dbfilename slave.rdb              # RDB文件名

slave-read-only yes               # 只读
```



## 04. 启动服务

```bash
yazi@sxyazi:~/redis $ redis-server ./redis.master.conf
yazi@sxyazi:~/redis $ redis-server ./redis.slave.conf
```



## 05. 查看状态

```bash
# Master
yazi@sxyazi:~/redis $ redis-cli
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:1
slave0:ip=127.0.0.1,port=6380,state=online,offset=99,lag=0
master_repl_offset:99
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2
repl_backlog_histlen:98

# Slave
yazi@sxyazi:~/redis $ redis-cli -p 6380
127.0.0.1:6380> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6379
master_link_status:up
master_last_io_seconds_ago:6
master_sync_in_progress:0
slave_repl_offset:99
slave_priority:100
slave_read_only:1
connected_slaves:0
master_repl_offset:0
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
```

