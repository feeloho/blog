# MySQL 主从配置

上篇文章，把 MySQL 服务环境搭建好了，这篇文章来写一下 MySQL 主从配置。



## 01. 修改 MySQL 配置文件

**my.master.cnf：**

```ini
[mysqld]
log-bin          = mysql-bin  # 二进制日志文件名
server-id        = 1          # 服务器ID
expire-logs-days = 7          # (可选) 日志有效期[天]
replicate-do-db  = test       # (可选) 要复制的数据库
binlog-ignore-db = mysql      # (可选) 忽略的数据库
binlog-ignore-db = information_schema
```

**my.slave.cnf：**

```ini
[mysqld]
log-bin                = mysql-bin   # (可选) 二进制日志文件名
server-id              = 2           # 服务器ID
replicate-do-db        = test        # (可选) 要复制的数据库
binlog-ignore-db       = mysql       # (可选) 忽略的数据库
replicate-ignore-table = mysql.user  # (可选) 忽略的表
read_only              = 1           # 只读
```



## 02. 启动主从服务

登入到 Master，执行：

```mysql
# 授权 yazi@192.168.2.103, 密码为 abcd1234
grant replication slave on *.* to 'yazi'@'192.168.2.103' identified by 'abcd1234';
flush privileges;
```

查看 Master 的状态

```mysql
show master status;

+------------------+----------+--------------+--------------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB         |
+------------------+----------+--------------+--------------------------+
| mysql-bin.000001 |      245 |              | mysql,information_schema |
+------------------+----------+--------------+--------------------------+
1 row in set (0.00 sec)
```

---

然后切换到 Slave，执行：

```mysql
change master to
master_host     = '192.168.2.103',    # Master的IP
master_user     = 'yazi',             # 刚才授权的账号
master_password = 'abcd1234',         # 刚才授权账号的密码
master_log_file = 'mysql-bin.000001', # 上面状态中的 `File` 字段
master_log_pos  = 245;                # 上面状态中的 `Position` 字段
```

启动 Slave

```mysql
start slave;
#stop slave  # 终止Slave
```

最后，查看状态是否正常

```mysql
show slave status \G

Slave_IO_Running:　Yes
Slave_SQL_Running:　Yes
```

