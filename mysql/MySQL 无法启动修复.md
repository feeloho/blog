# MySQL 无法启动修复

#### 报错如下: 

```
➜  bin ./mysqld
2017-03-23 08:42:32 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
2017-03-23 08:42:32 0 [Note] --secure-file-priv is set to NULL. Operations related to importing and exporting data are disabled
2017-03-23 08:42:32 0 [Note] ./mysqld (mysqld 5.6.34) starting as process 1358 ...
2017-03-23 08:42:32 1358 [Warning] Setting lower_case_table_names=2 because file system for /Applications/MAMP/db/mysql56/ is case insensitive
2017-03-23 08:42:32 1358 [Note] Plugin 'FEDERATED' is disabled.
2017-03-23 08:42:32 1358 [Note] InnoDB: Using atomics to ref count buffer pool pages
2017-03-23 08:42:32 1358 [Note] InnoDB: The InnoDB memory heap is disabled
2017-03-23 08:42:32 1358 [Note] InnoDB: Mutexes and rw_locks use GCC atomic builtins
2017-03-23 08:42:32 1358 [Note] InnoDB: Memory barrier is not used
2017-03-23 08:42:32 1358 [Note] InnoDB: Compressed tables use zlib 1.2.8
2017-03-23 08:42:32 1358 [Note] InnoDB: Using CPU crc32 instructions
2017-03-23 08:42:32 1358 [Note] InnoDB: Initializing buffer pool, size = 128.0M
2017-03-23 08:42:32 1358 [Note] InnoDB: Completed initialization of buffer pool
2017-03-23 08:42:32 1358 [Note] InnoDB: Highest supported file format is Barracuda.
2017-03-23 08:42:32 1358 [Note] InnoDB: The log sequence numbers 2782384 and 2782384 in ibdata files do not match the log sequence number 2782394 in the ib_logfiles!
2017-03-23 08:42:32 1358 [Note] InnoDB: Database was not shutdown normally!
2017-03-23 08:42:32 1358 [Note] InnoDB: Starting crash recovery.
2017-03-23 08:42:32 1358 [Note] InnoDB: Reading tablespace information from the .ibd files...
2017-03-23 08:42:32 1358 [ERROR] InnoDB: Attempted to open a previously opened tablespace. Previous tablespace mysql/slave_relay_log_info uses space ID: 3 at filepath: ./mysql/slave_relay_log_info.ibd. Cannot open tablespace shop/admin which uses space ID: 3 at filepath: ./shop/admin.ibd
2017-03-23 08:42:32 7fffad71c3c0  InnoDB: Operating system error number 2 in a file operation.
InnoDB: The error means the system cannot find the path specified.
InnoDB: If you are installing InnoDB, remember that you must create
InnoDB: directories yourself, InnoDB does not create them.
InnoDB: Error: could not open single-table tablespace file ./shop/admin.ibd
InnoDB: We do not continue the crash recovery, because the table may become
InnoDB: corrupt if we cannot apply the log records in the InnoDB log to it.
InnoDB: To fix the problem and start mysqld:
InnoDB: 1) If there is a permission problem in the file and mysqld cannot
InnoDB: open the file, you should modify the permissions.
InnoDB: 2) If the table is not needed, or you can restore it from a backup,
InnoDB: then you can remove the .ibd file, and InnoDB will do a normal
InnoDB: crash recovery and ignore that table.
InnoDB: 3) If the file system or the disk is broken, and you cannot remove
InnoDB: the .ibd file, you can set innodb_force_recovery > 0 in my.cnf
InnoDB: and force InnoDB to continue crash recovery here.
```



#### 修复方式:

It means one of your tables is corrupted. Weirdly this has happened to frequently upon a machine crash. This is how I fixed it:

1. open my.cnf (in MAMP > File > Edit Template > MySQL)
2. add the line: innodb_force_recovery = 1
3. save and start servers

The table will likely still be broken, but mysql should start. It means you may have to replace your table: wp_term_taxonomy

Note: If innodb_force_recovery = 1 does not work, try going up in numbers: innodb_force_recovery = 2 (etc) But tread carefully. Here's MySQL's warning on this setting:[https://dev.mysql.com/doc/refman/5.5/en/forcing-innodb-recovery.html](https://dev.mysql.com/doc/refman/5.5/en/forcing-innodb-recovery.html)

(参考链接: http://stackoverflow.com/a/40919026)



#### 另外补充:

查看MySQL配置文件路径方法:  `mysql --help | grep my.cnf`

```
➜  bin ./mysql --help | grep my.cnf
                      order of preference, my.cnf, $MYSQL_TCP_PORT,
/etc/my.cnf /etc/mysql/my.cnf /Applications/MAMP/conf/my.cnf ~/.my.cnf
```

若不存在创建一个文件即可.